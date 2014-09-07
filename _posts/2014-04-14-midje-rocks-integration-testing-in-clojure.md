---
layout: post
title:  "Midje Rocks! Integration Testing In Clojure"
date:   2014-04-14 16:23:15
categories: clojure midje testing
---
Clojure's great yeah- we write these tiny functions that do one thing well, and bury them in unit tests using our fave testing framework (mine's [Midje](https://github.com/marick/Midje)).  Then to keep things small and manageable, we compose our applications of micro-services, right?  So everything's awesome.  The problem is that we don't have an automated way of checking that all these services behave correctly together, so we don't test it properly.  Or worse, we test it by hand!

Clearly that's no good, we need a way of spinning up our services and testing them properly as part of our CI process, so we don't get called up at night when it all breaks.  Fortunately, Midje has excellent support for this built right in.  Let's start with a simple hello world Ring server, jazzed up a little bit:

{% highlight clojure %}

(ns it.core
  (:require [conf-er :refer [config]]
            [ring.adapter.jetty :refer [run-jetty]]))

(defn handler [request]
  {:status  200
   :headers {"Content-Type" "text/html"}
   :body    (config :body)})

(def server (atom nil))

(defn start-server
  "Start a ring server and store a reference"
  []
  (swap! server
         (fn [_] (run-jetty handler {:port 3000 :join? false}))))

(defn stop-server
  []
  (.stop @server))

{% endhighlight %}


We've gone a little bit further than the standard hello world example, because we're storing the ring server in an atom, so we can refer to it later and stop it.  You may also have noticed that we're using the very handy [conf-er](https://github.com/TouchType/conf-er) library to load the content that our server will be dishing out from the `:body` attribute of a config file called `application.conf`:


{% highlight clojure %}

{:body "hello world"}

{% endhighlight %}


Ah, there it is!  We're really pushing the boundaries of the web now.  Conf-er knows to use this file thanks to a JVM option we added in our `project.clj`:


{% highlight clojure %}

...
:jvm-opts ["-Dconfig=application.conf"]
...

{% endhighlight %}


This application can be run from a REPL simply by calling the `(start-server)` function, and shut down again with `(stop-server)`.  This isn't particularly exciting yet (I know!), but it allows us to use these functions in an integration test.  We'll use the `against-background` macro from Midje to fire up our application before our tests and shut it down after.  That means the test can be a real HTTP call to our app:


{% highlight clojure %}

(ns it.core-test
  (:use midje.sweet)
  (:require [clj-http.client :as client]
            [it.core :refer :all]))

(against-background [(before :contents (start-server))
                     (after :contents (stop-server))]

  (fact "hello world is served at root"
    (let [response (client/get "http://localhost:3000")]
      (response :status) => 200
      (response :body) => "just testing")))

{% endhighlight %}


We have to make the call in a let block because of this bug, but it's a pretty neat syntax for a test anyway.  The `against-background` macro will handle starting and stopping the server, and even copes when a fact throws an exception.  If we run this test now though, it'll fail - we'll get a 200 response, but it'll say "hello world" instead of "just testing". This is to show how we can use a lein profile to switch out our config file, to use this `test.conf` instead:


{% highlight clojure %}

{:body "just testing"}

{% endhighlight %}


Now let's add the profile into our project.clj:


{% highlight clojure %}

...
:jvm-opts ["-Dconfig=application.conf"]
:profiles {:test {:jvm-opts ["-Dconfig=test.conf"]}}
...

{% endhighlight %}


The new test profile overrides the JVM options we've previously specified, instructing conf-er to read the test file instead. We trigger the profile when we run our tests like this:

    lein with-profiles +test midje

The plus sign there adds the test profile (instead of replacing the existing profiles if we omit it).

This example is clearly a bit silly, but we could use the test config to fire the application up on a different port, change a database connection string to point to an in-memory test database, or change other environment-specific config.  By adding calls to other services in the before and after forms, we can ensure other parts of our application or third party services we rely on are running, and write tests that actually check the interactions between them.  We could write an integration test framework that's independent of the codebases of the individual components too, for a very neat separation of concerns.

Go forth and automate your integration testing!

You can find all the source code [here](https://github.com/conan/clojure-integration-testing).