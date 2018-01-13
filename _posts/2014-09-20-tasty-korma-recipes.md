---
layout: post
title:  "Tasty Korma Recipies"
date:   2014-09-20 16:00:00
redirect_from: /korma/clojure/sql/2014/09/20/tasty-korma-recipes.html
categories: korma clojure sql
comments: true
---

[Korma](http://sqlkorma.com/) is a handy library for SQL in Clojure; the [docs](http://sqlkorma.com/docs) are good, but sometimes an example is more useful.  Here are some handy things you can do (I've used MySQL but the techniques, like Korma, can be used with any relational database), using Korma v0.4.0:


## Prepare/Transform

Korma can automatically process data going in (via INSERT and UPDATE) and out (via SELECT) of the database, which you can use to make your life easier in the world of Clojure.

### Converting Timestamps

I like to use the most appropriate datatypes in my schemas, but often I want to use UNIX time in Clojure for working with timestamps:

{% highlight clojure %}

(ns conan.is
  (:import java.sql.Timestamp)
  (:require [clj-time.coerce :refer [from-long to-long]]
            [clj-time.format :refer [formatter parse unparse]]
            [korma.core :refer :all]
            [korma.db :refer :all]))

;; db connection defined here 

(def timestamp-formatter (formatter "yyyy-MM-dd HH:mm:ss"))

(defentity goals
  (prepare (fn [{timestamp :timestamp :as v}]
             (if timestamp
               (assoc v :timestamp (unparse timestamp-formatter (from-long timestamp)))
               v)))
  (transform (fn [{timestamp :timestamp :as v}]
               (if timestamp
                 (assoc v :timestamp (.getTime timestamp))
                 v))))

{% endhighlight %}

We define a formatter in the style of MySQL timestamps.  When we insert a row, we send the timestamp column as a long, in milliseconds since the epoch.  The `prepare` function turns the long into a [clj-time](https://github.com/clj-time/clj-time) `DateTime`, and then unparses that into a string using the formatter.  When selecting data, the `transform` function just calls the `getTime()` method of the `java.sql.Timestamp` that we get back from JDBC.  In Clojure we only ever have to work with milliseconds since the epoch, and in the database we can use the native datatypes.

### Avoiding Underscores

MySQL doesn't allow hyphens in table or column names, but in Clojure the convention is always to use hyphens in symbols.  Let's use `prepare` and `transform` to reconcile this.  Here's a table with underscores:

{% highlight sql %}

CREATE TABLE my_table (
  my_column INT
);

{% endhighlight %}

We can define an entity that will use hyphens instead of underscores:

{% highlight clojure %}

(defentity my-table 
  (table :my_table) 
  (prepare (fn [v] (rename-keys v {:my-column :my_column})))
  (transform (fn [v] (rename-keys v {:my_column :my-column}))))

{% endhighlight %}

We've simply renaming a specific column on the way in and out here, but we can generalise this to replace all the hyphens:

{% highlight clojure %}

(defentity my-table
  (table :my_table)
  (prepare
    (fn [v]
      (rename-keys
        v (reduce
            #(assoc %1 (first %2) (keyword (replace (name (first %2)) #"-" "_")))
            {} v))))
  (transform
    (fn [v]
      (rename-keys
        v (reduce
            #(assoc %1 (first %2) (keyword (replace (name (first %2)) #"_" "-")))
            {} v)))))

{% endhighlight %}

Here we're taking the data map and running [`clojure.set/rename-keys`](http://clojuredocs.org/clojure_core/clojure.set/rename-keys) over it, passing in a map describing how to replace underscores with hyphens that we've created by using [`clojure.string/replace`](http://clojuredocs.org/clojure_core/clojure.string/replace).  Now we can avoid using underscores in the entity and column names everywhere:

{% highlight clojure %}

(insert my-table (values {:my-column 1}))

(select my-table)

;; => {:my-column 1}

{% endhighlight %}

Note that when you insert a row into a table with an auto-generated primary key, Korma returns a map containing the `:generated_key` for that row.  This now gets transformed to `:generated-key`.

## Join Types 

You can specify the join type with a keyword:

{% highlight clojure %}

(select users (join :inner orders (= :users.id :orders.user_id))

{% endhighlight %}


## Table Aliases

It's useful to alias columns in a select like this:

{% highlight clojure %}

(select users (fields [:fullname :name]))

{% endhighlight %}

You can do the same thing with your entity names, which is handy for joins.  These two are the same:

{% highlight clojure %}

(select users 
  (fields :users.name :orders.product) 
  (join :inner orders  (= :users.id :orders.user_id)))

(select [users :u] 
  (fields :u.name :o.product) 
  (join :inner [orders :o]  (= :u.id :o.user_id)))

{% endhighlight %}
