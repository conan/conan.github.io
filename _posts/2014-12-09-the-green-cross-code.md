---
layout: post
title:  "The Green Cross Code: Parallel Selenium In Clojure"
date:   2014-12-09 16:00:00
categories: webdriver selenium testing clojure
comments: true
---

I trust my co-developers to write good code.  I trust my open-source benefactors to do their best to make their code
work.  I do not, however, trust browser manufacturers to be compatible with the great things these people make.  When
you can't trust what someone says, you demand proof - and in the web world, that means cross-browser testing.

We use the excellent [Sauce Labs](https://saucelabs.com) to provide us with a [large](https://saucelabs.com/platforms)
number of OS/browser/version combinations that we can use to check our code behaves correctly everywhere (we're
currently testing on around 90 different environments).  As a Clojure shop though, we found that
[clj-webdriver](https://github.com/semperos/clj-webdriver) was the only decent option out there,
but the remote interface in that library
[has been ported from Selenium Grid](https://github.com/semperos/clj-webdriver/issues/99) and doesn't expose the
functionality to configure remote environments correctly (the
[DesiredCapabilities](https://selenium.googlecode.com/git/docs/api/java/org/openqa/selenium/remote/DesiredCapabilities.html)
object).  We didn't have time to fix this problem generally, so instead we made a lightweight library for parallelising
remote tests using WebDriver in Clojure: [Examinant](https://github.com/BrightNorth/examinant).

Examinant is aimed at Clojure developers familiar with WebDriver.  It handles concurrent execution of selenium tests
on a remote service, and makes a best effort at aggregating these in a clojure.test suite; neither clojure.test nor
midje currently support parallel execution of tests, so unfortunately all your tests have to run in a single `deftest`,
but Examinant will give you useful output and pass/fail status.

Browser configurations can be defined using whatever format your remote provider expects, this is how we set things up
with Sauce Labs:

{% highlight clojure %}
(def browser-specs [{:browserName "chrome" :version "38" :platform "Windows 8.1"}
                    {:browserName "safari" :version "8" :platform "OS X 10.10"}
                    {:browserName "android" :version "4.4" :platform "LINUX"
                     :device-orientation :portrait :deviceName "Google Nexus 7 HD Emulator"}
                    {:browserName "iPhone" :version "8.1" :platform "OS X 10.9"
                     :device-orientation :portrait}])
{% endhighlight %}

Each of your tests should be defined as Clojure functions taking a single argument - a WebDriver object.  They can
contain clojure.test assertions just like your regular `deftest`s; it's not necessary to include the type hints, but
I've left them in for clarity:

{% highlight clojure %}
(defn google-logo
  "Checks whether the google logo has the correct title"
  [^RemoteWebDriver driver]
  (.get driver "http://www.google.co.uk")
  (let [^WebElement logo-div (.findElementById driver "hplogo")
        title (.getAttribute logo-div "title")]
    (is (= title "Google"))))

(def tests [google-logo])
{% endhighlight %}

The entry point to Examinant is the `remote-tests` function, which takes 4 arguments:

* the url of your remote provider (which can contain credentials, as in `http://user:password@provider.com`)
* a sequence of browser specifications (like `browser-specs` above)
* a sequence of test functions (like `tests` above)
* an optional concurrency count, which determines the number of concurrent threads that will be used.  If not supplied,
the [default Clojure agent send-off thread pool](http://clojure-doc.org/articles/language/concurrency_and_parallelism.html#using-custom-executors-with-agents)
will be used.

{% highlight clojure %}
(deftest remote-google-logo-tests
  (examinant/remote-tests url browser-specs tests 8))
{% endhighlight %}

Examinant will handle aggregating the results of your tests for you, and will output useful information in the
clojure.test style in the case of a failure; note that because we're aggregating multithreaded test results, there's a
hack in use: instead of the regular "expected" value you'll see a nil check, but the individual failure in your
test will still be visible in the "actual" value, and the test results will be correct.  Hey, I'm open to Pull Requests...

The aim was to write a library that was very easy to use for a specific purpose: running large numbers of remote tests
concurrently, using the plain WebDriver API.  We haven't attempted to wrap WebDriver in idiomatic Clojure like you'll
find in the [Taxi API](https://github.com/semperos/clj-webdriver/wiki/Taxi-API-Documentation), but there is one useful
function that's provided to make things easier: `wait-until`.  This is such a common operation and is awkward to write
in Clojure using Java interop with
[WebDriverWait](https://selenium.googlecode.com/git/docs/api/java/org/openqa/selenium/support/ui/WebDriverWait.html), so
we rolled one ourselves; give it a WebDriver instance and a predicate function, and it'll keep polling the function
until it returns true; if an optional timeout is specified, it'll give up after that.  This is essential for ensuring
that a document is in the state you want it to be before running your tests, especially when running against tens or
hundreds of different environments.

I hope you're inspired to be more brutal in your testing; the Examinant will keep your scripts in order.
