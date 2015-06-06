---
layout: post
title:  "Freedom Fighting: A Liberator Primer"
date:   2015-06-05 19:00:00
categories: clojure liberator rest
comments: true
---

If you've ever built a webservice in Clojure, you've done it in [Ring](https://github.com/ring-clojure).  Writing 
HTTP responses is boring, but fortunately [Liberator](https://clojure-liberator.github.io/liberator/github.html) is here
to help. It provides a bunch of boilerplate that makes it easy to reason about the HTTP pipeline, and can make webapps 
really easy to read and test. This post generally assumes you've used Liberator a bit, but if not don't worry - the 
[docs](https://clojure-liberator.github.io/liberator/) are excellent, and you should familiarise yourself with the
[Decision Graph](https://clojure-liberator.github.io/liberator/tutorial/decision-graph.html) before reading further.

It can take a  bit of experience (read: anger and frustration) to get the most out of Liberator;
luckily I've done the necessary and can save you some time.  Free your mind and the REST will follow.

# 1. Returning representations before Content-Type negotiation

Before a request has made it to the `:media-type-available` decision function, Liberator doesn't know what media type
it should return when building responses.  Unfortunately, that decision comes after other decisions in which you're 
likely to want to return content in responses, such as `:malformed` and `:authorized`.  In order for Liberator to build
a Ring response for you in the usual way, you have to help it out a bit by explicitly stating the media type you want
it to use:

{% highlight clojure %}
(defresource cars
  ...
  :malformed? (fn [ctx] (when-let [error (car-is-malformed? ctx)]
                          {:error error :representation {:media-type "application/json"}}))
{% endhighlight %}

Of course, if the request is malformed then you might not be able to identify a good media type to send, 

# 2. Falling back to Ring
 
Liberator will coerce data structures you return from handlers into Ring responses (using its `as-response` function), 
but sometimes you need to manipulate the response a bit before pushing it out, for example to add custom headers.  As of 
Liberator 0.13, you can give the `ring-response` function an extra map of stuff that'll get merged in before the 
coercion happens. We often use this to return 409 (Conflict) for POST requests that try to create something (typically
a new user account) that already exists:

{% highlight clojure %}
(defresource users
  :allowed-methods [:post]
  ...
  :handle-created (fn [ctx]
                    (if-let [conflict (:conflict ctx)]
                      (ring-response {:error conflict} {:status 409})
                      {:name "MacLeod"}))
{% endhighlight %}

In this example, the usual response status would be 201, because that's what gets added to a response by 
`:handle-created`, but we override that in the case that some earlier decision function has put a `:conflict` key on
the context with an error message in it to return a 409, which Liberator 
[doesn't currently support for POST](https://github.com/clojure-liberator/liberator/issues/107)s.

There's a good example of adding a custom header in the 
[Altering a generated response](https://clojure-liberator.github.io/liberator/doc/representations.html) section of the
Liberator docs too.

# 3. Handling Exceptions

Liberator will catch any exceptions that are thrown by your handlers, and silently swallow 'em whole whilst returning
a 500. Quite often you want to see what went wrong - you can plumb in a `handle-exception` handler that can do something
useful instead.  Here's one that logs:

{% highlight clojure %}
(defn handle-exception
  "Log exceptions in Liberator at ERROR level."
  [ctx]
  (let [e (:exception ctx)]
    (error e "Liberator caught" (.getClass e) "message:" (.getMessage e))))
{% endhighlight %}

You'd use this as your `:handle-exception` handler function:

{% highlight clojure %}
(defresource my-resource
  ...
  :handle-exception handle-exception)
{% endhighlight %}

# 4. Resource defaults

Many of your resources will have things in common - for example, they all return the same media type.  You can define
a map of defaults that will be merged with any `defresource` it gets passed to, to keep things DRY.   This is a good 
place for that exception handler too:

{% highlight clojure %}
(def resource-defaults
  {:available-media-types ["application/json"]
   :handle-exception      handle-exception})
{% endhighlight %}

# 5. Tracing
