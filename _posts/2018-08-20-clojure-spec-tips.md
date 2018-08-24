---
layout: post
title: "Clojure Spec Tips"
date: 2018-08-20 20:00:00
categories: spec clojure
comments: true
published: true
---

This post includes some practical tips for getting the most out of spec.  I'll update it whenever I find out something new, so please feel free to tell me things I don't know.

# Project setup

## Clojure

I always have a default namespace that loads when I start my REPL (using [`:init-ns`](https://github.com/technomancy/leiningen/blob/master/sample.project.clj#L369)), and I run a couple of side effects when it loads.

[expound]() formats your spec output better:

``` clojure
(alter-var-root #'s/*explain-out* (constantly expound/printer))
```

We write function specs for all our functions, so we want them instrumented; [orchestra](https://github.com/jeaye/orchestra) is a drop-in replacement for `clojure.spec.test` which instruments your `:ret` and `:fn` specs (I alias it as `stest`) in addition to the `:args`:

``` clojure
(stest/instrument)
```

With those two you'll fail much faster. If you're using a [reloaded](http://thinkrelevance.com/blog/2013/06/04/clojure-workflow-reloaded) workflow that calls [`repl/refresh`](https://github.com/clojure/tools.namespace#reloading-code-usage), you'll need to re-instrument all your functions after that; this can be noisy, so I hide the noise with a hack:

``` clojure
(defn reset
  []
  ;; do some reloaded stuff here
  (repl/refresh)
  (with-out-str (stest/instrument)))
```

## Clojurescript

### Dev tools

Spec errors can be difficult to read in the console, so if you're using Chrome you'll want [cljs-devtools](https://github.com/binaryage/cljs-devtools).  They explain [how to install](https://github.com/binaryage/cljs-devtools/blob/master/docs/installation.md) better than I can, but don't forget to [enable custom formatters](https://github.com/binaryage/cljs-devtools/blob/master/docs/installation.md#enable-custom-formatters-in-chrome) in Chrome (we keep a note about this in our dev setup instructions in our readme because it costs so much time for new devs if they miss it).

### Setting up

I run this function when my application starts, and also use it as the `:on-jsload` property in my cljsbuild map:

``` clojure
;; in "dev" build config
:figwheel {:on-jsload "my-ns.core/dev-setup"}
```
``` clojure
(defn dev-setup []
  (when goog.DEBUG
    (do
      (set! s/*explain-out* expound/printer)
      (enable-console-print!)
      (js/console.warn 
        "Debug mode, to see logs please set your inspector to show Verbose logging")
      (stest/instrument))))
```

You can see that we're setting expound as the printer for spec errors and instrumenting our functions - the `stest` alias refers to orchestra in exactly the same was as above. `goog.DEBUG` is a Google Closure property that's only set to `true` in my dev build. Note that this won't get eliminated as dead code in production builds. 

### Re-frame

If you're using [re-frame](https://github.com/Day8/re-frame) you can get some sweet db validation with this interceptor:

``` clojure
(defn validate
  "Returns nil if the db conforms to the spec, throws an exception otherwise"
  [spec db]
  (when goog.DEBUG
    (when-let [error (s/explain-data spec db)]
      (throw (ex-info 
               (str "DB spec validation failed: " 
                    (expound/expound-str spec db)) 
               error)))))

(def validate-interceptor (rf/after (partial validate :my/db)))
```

Then hook it in to every event handler (it's boilerplate, but it's worth it; you could always write a macro to avoid it):

``` clojure
(rf/reg-event-db :some/event
  [validate-interceptor]
  (fn [db [event]]
    ;; handle event
    ))
```

This will blow up every time one of your handlers attempts to update the app-db with invalid data.

# When to use collections, maps and regex

* [`coll-of`](https://clojuredocs.org/clojure.spec.alpha/coll-of) is for collections
* regex ([`cat`](https://clojuredocs.org/clojure.spec.alpha/cat), [`+`](https://clojuredocs.org/clojure.spec.alpha/+), [`*`](https://clojuredocs.org/clojure.spec.alpha/*), [`?`](https://clojuredocs.org/clojure.spec.alpha/_q), [`&`](https://clojuredocs.org/clojure.spec.alpha/&)) are for semantic placing, i.e. when a certain thing has to be in a certain place.  don't default to `(s/* ...)` for collections, or you'll end up with weird exceptions later when you do [`stest/check`](https://clojure.github.io/spec.alpha/clojure.spec.test.alpha-api.html#clojure.spec.test.alpha/check) or [`s/exercise-fn`](https://clojuredocs.org/clojure.spec.alpha/exercise-fn) and similar.
* [`map-of`](https://clojuredocs.org/clojure.spec.alpha/map-of) is for maps, when you know the _types_ of the keys but not the actual keys themselves

# Maps

## Naming conventions for nested maps

I find it useful to have a predictable structure to my data.  This can be achieved by ensuring each map has its own set of properties, and that the hierarchy is modelled in the namespaces:

``` clojure
(s/def :company.department.employee/name string?)
(s/def :company.department.employee/payroll-number nat-int?)
(s/def :company.department/employee 
  (s/keys :req [:company.department.employee/name
                :company.department.employee/payroll-number]))
(s/def :company.department/employees (s/coll-of :company.department/employee))
(s/def :company.department/id uuid?)
(s/def :company/department (s/keys :req [:company.department/id
                                         :company.department/employees]))
```        
This approach is verbose but there are no surprises, which keeps things nice and simple.  There are as many different ways to model data as there are domains, but this one is a useful default if you're unsure.

## Merging map specs

You can use `s/merge` to merge map specs:

``` clojure
(s/def :car.engine/petrol (s/keys :req [:car.engine.petrol/miles-per-gallon]))
(s/def :car.engine/electric (s/keys :req [:car.engine.electric/time-to-charge]))
(s/def :car.engine/hybrid (s/merge :car.engine/petrol
                                   :car.engine/electric))
```        

## Empty map specs

Using [`s/keys`](https://clojuredocs.org/clojure.spec.alpha/keys) is asking spec to check the value of every key in the map that has a matching spec. If you don't specify `:req` or `:opt`, that's all it will do. It isn't possible to exclude keys from a map using spec, so you can't actually specify that a map must be empty.

``` clojure
(s/def ::marmite #{true false})
(s/def :unknown/keys (s/keys)) ;; no mention of what should be in here
(s/explain :unknown/keys {::marmite "yum!"})
In: [:user/marmite] val: "yum!" fails spec: :user/marmite at: [:user/marmite] predicate: #{true false}
```

# Exclusive or (XOR)

You can specify keys using a boolean OR:

``` clojure
(s/def :my.popcorn.type/salty? boolean)
(s/def :my.popcorn.type/sweet? boolean)
(s/def :my/popcorn (s/keys :req [:my.popcorn/kernel
                                 (or :my.popcorn.type/salty?
                                     :my.popcorn.type/sweet?)]))
```

There's a problem here though: the OR is not exclusive, so this spec will allow my popcorn to be both salty AND sweet.  Crazy, right?  What's even worse is that the generator for this will definitely produce mixed popcorn, leading to bugs.  Nobody wants bugs in their popcorn.

We can fix this madness by using [`s/or`](https://clojuredocs.org/clojure.spec.alpha/or), which is an exclusive OR (XOR):

``` clojure
(s/def :popcorn/type (s/or (s/keys :req [:my.popcorn/kernel :popcorn.type/salty?])
                           (s/keys :req [:my.popcorn/kernel :popcorn.type/sweet?])))
```

This unfortunately does involve duplication, but if our specs are large we could solve that by using [`s/merge`](#merging-map-specs). Simple over easy.

# Functions

## Function arguments

Function arguments are positional; to define specs for them we can use [regex specs](https://clojure.org/guides/spec#_sequences):

``` clojure
(s/fdef clojure.core/get
  :args (s/cat :m (s/keys) :k keyword?))
```

We could also use a [tuple](https://clojuredocs.org/clojure.spec.alpha/tuple):

``` clojure
(s/fdef clojure.core/get
  :args (s/tuple (s/keys) keyword?))
```

If we expect lots of the same thing (so don't care about position) we can use [`coll-of`](https://clojuredocs.org/clojure.spec.alpha/coll-of):

``` clojure
(s/fdef clojure.core/+
  :args (s/coll-of number?))`
```
  
## Zero-arity functions

Spec an empty `:args` list with [`(s/cat)`](https://clojuredocs.org/clojure.spec.alpha/cat) because we aren't allowed to create an empty [`tuple`](https://clojuredocs.org/clojure.spec.alpha/tuple):

``` clojure
(s/fdef clojure.core/clojure-version
  :args (s/cat)))
```

# Datomic 

## Connections and databases
``` clojure
(s/def :datomic/db #(s/or :datomic (instance? Database %)
                          :datoms (s/coll-of vector? :kind vector?)))
(s/def :datomic/conn #(instance? Connection %))
```
These can be used for function specs that take a database or connection as an argument.  Note that the database spec allows for a vector of datoms, so in our tests we can pass in test data to pure functions without having to get an actual database value.

## Specs & schema

The [naming conventions for nested maps](#naming-conventions-for-nested-maps) translate perfectly to Datomic [schema](https://docs.datomic.com/on-prem/schema.html):

``` clojure
{:db/migration-2018-02-23_departments-and-employees
    {:txes [[{:db/doc "Department unique ID"
            :db/ident :company.department/id
            :db/valueType :db.type/uuid
            :db/cardinality :db.cardinality/one
            :db/unique :db.unique/identity}
            {:db/doc "The employees belonging to a department"
            :db/ident :company.department/employees
            :db/valueType :db.type/ref
            :db/cardinality :db.cardinality/many}
            {:db/doc "An employee's full name"
            :db/ident :company.department.employee/name
            :db/valueType :db.type/string
            :db/cardinality :db.cardinality/one}
            {:db/doc "An employee's payroll number"
            :db/ident :company.department.employee/payroll-number
            :db/valueType :db.type/long
            :db/cardinality :db.cardinality/one
            :db/unique :db.unique/identity}]]}}
```            

This way you can provide entity validation on the way in and out of Datomic, and easily populate a test database with generated data using [`gen/sample`](https://clojure.github.io/spec.alpha/clojure.spec.gen.alpha-api.html#clojure.spec.gen.alpha/sample) or [`s/exercise`](https://clojuredocs.org/clojure.spec.alpha/exercise). 

A useful heuristic is this:

* If a `ref` has the `isComponent` property, add another level to the namespace hierarchy
* If a `ref` has a unique identifier instead, reset back to the top of the hierarchy

## Pull patterns

You can make a pull pattern for Datomic's [Pull API](https://docs.datomic.com/on-prem/pull.html) by parsing the spec you've defined using the [`s/form`](https://clojuredocs.org/clojure.spec.alpha/form) function to return the spec as data (a vector), which can then be destructured:
``` clojure
(defn pattern
    "Gets a Datomic Pull pattern for the given spec"
    [spec]
    (let [[_ _ req _ opt] (s/form spec)]
    (vec (concat req opt))))
    
(s/fdef pattern
    :args (s/cat :spec keyword?)
    :ret vector?)
```

It's messy but it works, although if we want nested attributes (for example, if we wanted to pull an entire department and all its employee data as well) we'd have to write a richer recursive function. Why do this? It's useful for removing `:db/id`s and other sensitive values that would be returned by using the `[:*]` pattern, without repeating ourselves by writing specs and pull patterns.

We can now get the pattern for an arbitrary entity spec and use it in a query:

``` clojure
(defn employee-by-payroll-number
    [payroll-number]
    (let [employee-pattern (pattern :company.department/employee)]
    (d/q '[:find (pull ?e pattern) .
            :in $ ?pn pattern
            :where [?e :company.department.employee/payroll-number ?pn]
            db payroll-number employee-pattern)))

(s/fdef employee-by-payroll-number
    :args (s/cat :payroll-number :company.department.employee/payroll-number)
    :ret (s/nilable :company.department/employee))
```
