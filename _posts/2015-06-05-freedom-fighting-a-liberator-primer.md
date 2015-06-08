---
layout: post
title:  "Freedom Fighting: A Liberator Primer"
date:   2015-06-05 19:00:00
categories: clojure liberator rest
comments: true
---

If you've ever built a webservice in Clojure, you've done it using [Ring](https://github.com/ring-clojure/ring).  
Writing HTTP responses is boring, but fortunately [Liberator](https://clojure-liberator.github.io/liberator/) 
provides a bunch of boilerplate that makes it easy to do and easy to read and test. This post generally assumes you've 
used Liberator a bit, but if not don't worry - the [docs](https://clojure-liberator.github.io/liberator/tutorial/) are excellent, but you should familiarise yourself with the
[Decision Graph](https://clojure-liberator.github.io/liberator/tutorial/decision-graph.html) before reading further.

Liberator is a time-saver for sure, but it can take a  bit of experience (read: anger and frustration) to get the most 
out of it; luckily I've done the necessary and have some handy tips to save you time.  Free your mind and the REST will 
follow.


# 1. Handling Exceptions

Liberator will catch any exceptions that are thrown by your handlers, and swallow 'em whole whilst returning a 500. Quite often you want to see what went wrong, though.  You can get the exception from the context and plumb in a `handle-exception` handler that can do something useful instead.  Here's one that logs:

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


# 2. Resource defaults

Many of your resources will have things in common - for example, they all return the same media type.  You can define
a map of defaults that will be merged with any `defresource` it gets passed to, to keep things DRY.   This is a good 
place for that exception handler too:

{% highlight clojure %}
(def resource-defaults
  {:available-media-types ["application/json"]
   :handle-exception      handle-exception})
   
(defresouce my-resource resource-defaults
  ...
  :handle-ok ...)
{% endhighlight %}


# 3. Returning representations before Content-Type negotiation

Before a request has made it to the `:media-type-available` decision function, Liberator doesn't know what media type
it should return when building responses.  Unfortunately, that decision comes after others in which you're likely to 
want to return content in responses, such as `:malformed` and `:authorized`.  In order for Liberator to build a Ring 
response for you in the usual way, you have to help it out a bit by explicitly stating the media type you want it to 
use for the representation:

{% highlight clojure %}
(defresource cars
  ...
  :malformed? (fn [ctx] (when-let [error (car-is-malformed? ctx)]
                          {:error          error 
                           :representation {:media-type "application/json"}}))
{% endhighlight %}

Of course, if the request is malformed then you might not be able to identify a good media type to send. You could manually interrogate the request, but I often find I just want to use JSON as the default anyway.


# 4. Falling back to Ring
 
Liberator will coerce data structures you return from handlers into Ring responses (using its `as-response` function), 
but sometimes you need to manipulate the response a bit before pushing it out, for example to add custom headers.  The `ring-response` function lets you build a response manually, but Liberator 0.13 introduced a feature to override parts of a generated response whilst still using Liberator's coercion:

{% highlight clojure %}
(defresource users
  :allowed-methods [:post]
  :post! (fn [ctx]
           (if (user-exists? ctx)
             {:conflict "A user with that email address already exists"}
             (create-user! ctx)
  :handle-created (fn [ctx]
                    (if-let [conflict (:conflict ctx)]
                      (ring-response {:error conflict} {:status 409})
                      {:name "MacLeod"}))
{% endhighlight %}

In this example, we're handling a POST request to create a new user.  The usual response status would be 201 (Created), 
because that's what gets added to a response by `:handle-created`, but we override that if the user already existed. The 
`:post!` function puts a `:conflict` key on the context. We can check for this and return a 409 (Conflict), which 
Liberator [doesn't currently support for POST](https://github.com/clojure-liberator/liberator/issues/107), along with some details of the error.

There's a good example of adding a custom header in the 
[Altering a generated response](https://clojure-liberator.github.io/liberator/doc/representations.html) section of the
Liberator docs too.


# 5. Tracing

The Liberator docs on [debugging](https://clojure-liberator.github.io/liberator/doc/debugging.html) explain how to use
`wrap-trace`, and how to use the `log!` function to add your own debugging output in as well.  One thing that's 
particularly nice is that if you use the `:header` and `:ui` options, you'll get a Link header that contains a relative 
URL.  That URL points to Liberator's web interface, and it'll display the
[Decision Graph](https://clojure-liberator.github.io/liberator/tutorial/decision-graph.html) with the execution flow of
the last request highlighted, which is really handy for debugging badly behaving resources.
