---
layout: post
title: "A spec for URLs in Clojure"
date: 2018-01-13 20:00:00
categories: spec clojure
comments: true
published: true
---

Let's get some [clojure.spec]():
``` clojure
(require '[clojure.spec.alpha :as s]
         '[clojure.spec.gen.alpha :as sgen])
```         

Spec has built-in specs for a number of types, including URLs:

``` clojure
(s/valid? uri? "http://conan.is")
=> true
```

Many of these also include generators:

``` clojure
  (sgen/sample (s/gen uri?) 3)
  => []
```

This generator isn't very good - the URLs it generates are valid, but they're all pretty similar. We need a more comprehensive generator that will produce a wide variety of URLs; after all, variety is the spice of generative testing. 

URLs consist of many parts: 

* protocol
* username 
* password
* host
* port
* path
* query
* anchor

Your average URL won't contain them all, but we want our tests to be as comprehensive as possible. 

## Component generators

Let's write our own generator for URLs, and use it in an improved URL spec. A generator is a no-arg function that returns a [`clojure.test.check.generators.Generator`](https://clojure.github.io/test.check/clojure.test.check.generators.html#var--.3EGenerator).

To construct URLs we can use Chas Emerick's handy [url](https://github.com/cemerick/url) library:

``` clojure
(require '[cemerick.url :as url])
```

It features a handy constructor for URLs, `->URL`, which takes a vector of the above components and gives us back a [`java.net.URI`](https://docs.oracle.com/javase/8/docs/api/java/net/URI.html) object (Types? Constructors? Objects? OOPs!). We'll need a generator for each of them, but first let's set ourselves up with a generator that makes non-empty alphanumeric strings - we'll be needing a lot of these later:

``` clojure
(defn non-empty-string-alphanumeric
  []
  (sgen/such-that #(not= "" %) 
    (sgen/string-alphanumeric)))

(sgen/generate (non-empty-string-alphanumeric))
=> "j1cf5jDg6toLyP"
```

The [`fmap`](https://clojure.github.io/spec.alpha/clojure.spec.gen.alpha-api.html#clojure.spec.gen.alpha/fmap) in the first generator here lets us apply a function to the result of a generator, in this case to url-encode the string.

The [`such-that`](https://clojure.github.io/spec.alpha/clojure.spec.gen.alpha-api.html#clojure.spec.gen.alpha/such-that) in the second generator takes a predicate and another generator, and filters that generator's output using the predicate. This does mean that some of the generated values are thrown away, which is wasted work, but we just want to filter out the empty strings, and they're rare compared to all possible outputs of `string-alphanumeric`. 

### Protocol

This is easy, we just want to allow `http` and `https` for now, so we can use [`elements`](https://clojure.github.io/spec.alpha/clojure.spec.gen.alpha-api.html#clojure.spec.gen.alpha/elements) to select randomly from a set:

``` clojure
(sgen/elements #{"http" "https"})
```

### Username, password, host

These are just strings, and surprisingly they can be empty. We can use `url-encoded-string-alphanumeric`.


### Port

Ports can be any integer from 1 to 65535, and the [`choose`](https://clojure.github.io/test.check/clojure.test.check.generators.html#var-choose) function lets us create generators which select from a range of numbers:

``` clojure
(sgen/choose 1 65535)
```

### Path

This is where things get interesting. The path section of a URL is a sequence of non-empty strings separated by forward slashes. We can make more complex generators like this by combining simpler ones. 

Now that we have a generator that can create non-empty strings, we can easily create non-empty vectors of non-empty url-encoded strings, and separate them with slashes:

``` clojure
(sgen/fmap #(->> %
                (interleave (repeat "/"))
                (apply str))
  (sgen/not-empty
    (sgen/vector
      (non-empty-url-encoded-string-alphanumeric))))
```

Starting from the bottom, we're using our `non-empty-url-encoded-string-alphanumeric` generator and passing it to [`vector`](https://clojure.github.io/test.check/clojure.test.check.generators.html#var-vector) to get a vector of non-empty strings.  We're ensuring the vector itself is [`not-empty`](https://clojure.github.io/test.check/clojure.test.check.generators.html#var-not-empty).  Finally we're applying a function to the generated vector using [`fmap`](https://clojure.github.io/spec.alpha/clojure.spec.gen.alpha-api.html#clojure.spec.gen.alpha/fmap) which interleaves the strings with forward slashes, and gives us back the whole lot as a single string.

### Query

To generate query params, we need a map of random keys and values, which we can create using [`map`](https://clojure.github.io/test.check/clojure.test.check.generators.html#var-map). IT takes two generators, one for the keys and one for the values, and a map of options:

``` clojure
(sgen/map
  non-empty-url-encoded-string-alphanumeric
  non-empty-url-encoded-string-alphanumeric
  {:max-elements 10}) 
```

### Anchor

Anchors can be empty, so we use `url-encoded-string-alphanumeric`.

## URL generator

Here's how our full URL generator looks:

``` clojure
(def url-gen
  "Generator for generating URLs; note that it may generate http URLs 
  on port 443 and https URLs on port 80, and only uses alphanumerics"
  (sgen/fmap 
    (partial apply (comp str url/->URL))
    (sgen/tuple
      ;; protocol
      (sgen/elements #{"http" "https"})
      ;; username
      url-encoded-string-alphanumeric
      ;; password
      url-encoded-string-alphanumeric
      ;; host
      url-encoded-string-alphanumeric
      ;; port
      (sgen/choose 1 65535)
      ;; path
      (sgen/fmap #(->> %
                       (interleave (repeat "/"))
                       (apply str))
        (sgen/not-empty
          (sgen/vector
            non-empty-url-encoded-string-alphanumeric)))
      ;; query
      (sgen/map
        non-empty-url-encoded-string-alphanumeric
        non-empty-url-encoded-string-alphanumeric
        {:max-elements 3})
      ;; anchor
      url-encoded-string-alphanumeric)))
```

We've used [`tuple`](https://clojure.github.io/test.check/clojure.test.check.generators.html#var-tuple) to generate the vector of elements we need using each of our generators, and passed that vector to our URL constructor. As the last step we turn it into a string.

We can now generate random URLs:

``` clojure
(sgen/generate url-gen)
=> "https://S5xusj6zS:Up9S786kGF@1QK4956802NQuGZE4vgq7Q5w689v:61603/a4jWg250687kTTS9iA3FCXLKbxT1/5aa2Pzlg0Xg9Z5gFd22v09r3/507Q838m1513t339sXeCYuhSU2RV/63HP3s0Lw9BeTgDL7?u0X1VI4hPy7P392yY8Jn4e9L394lg4=LRiDy9zLyi6MBb2J#"
```

## URL Spec

We can now create a spec using our URL generator:

``` clojure
(s/def ::url (s/with-gen
                 (s/and string? #(try (url/url %) (catch Throwable t false)))
                 (fn [] url-gen)))
```

[`with-gen`](https://clojure.github.io/spec.alpha/clojure.spec.alpha-api.html#clojure.spec.alpha/with-gen) takes a spec and a no-args function that returns a generator (for why?). Our spec has a short circuit to check we're dealing with a string, and then tries to construct a URL from it, failing if an exception is thrown. 

We can now validate URLs:

``` clojure
(s/valid? ::url "http://conan.is")
=> true
```

You can find the code [here](https://gist.github.com/conan/8f0c879c47d14d5713f7a0986f81285d).

Neat!