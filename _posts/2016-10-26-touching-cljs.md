---
layout: post
title: "Touching ClojureScript"
date: 2016-10-28 13:30:00
categories: clojurescript hammer-js
comments: true
custom_js:
- zoomer
---

You can [touch it yourself](/zoomer).  

As a [Windows user](http://conan.is/clojure/clojurescript/windows/2015/10/05/clojure-on-windows.html), I'm used to touching my laptop screen, and most of your users today are probably used to fingering, rather than clicking, the internet on their phones or tablets.  [Hammer.js](http://hammerjs.github.io/) is a frictionless JavaScript library for working with touch gestures, and ClojureScript's interop makes it smooth to use. Here we'll add pinch-to-zoom support to an image. 

Hammer uses things called Recognisers to listen for (feel?) touch gestures.  We'll be using three of them: [pinch](http://hammerjs.github.io/recognizer-pinch/), [pan](http://hammerjs.github.io/recognizer-pan/) and [tap](http://hammerjs.github.io/recognizer-tap/).  We'll need to make and configure an instance of [Hammer.Manager](http://hammerjs.github.io/api/#hammer.manager), and the same for each of the three Recognisers; then we'll plug them together, and finally we'll add some handler functions to do the actual work.


# Creating the Zoomer

In this code I'm trying out [Peter Taoussanis](https://www.taoensso.com/)' [naming conventions](https://github.com/ptaoussanis/encore/blob/master/src/taoensso/encore.cljx#L12) from his [encore](https://github.com/ptaoussanis/encore) library.  Hopefully it's all clear.


## Dependencies

We don't need much for this, just Hammer and [Reagent](https://reagent-project.github.io/) (you could just as easily use [Om](https://github.com/omcljs/om), [Quiescent](https://github.com/levand/quiescent), [Rum](https://github.com/tonsky/rum) or some other React wrapper):

{% highlight clojure %}
(ns zoomer.core
  (:require [cljsjs.hammer]
            [reagent.core :as r]))
{% endhighlight %}              


## The Zoomer component

Let's start by creating a new Reagent component, which contains an image:

{% highlight clojure %}
(defn zoomer
  []
  (let [!hammer-manager (r/atom nil)
        !zoom           (r/atom {:x 0 :y 0 :scale 1})
        !start-zoom     (r/atom {:x 0 :y 0 :scale 1})]
    (r/create-class
      {:reagent-render
       (fn [_]
         [:div.zoomer
          [:img {:src   "/images/clojure-logo.png"
                 :style (transform @!zoom)}]])})))              
{% endhighlight %}                 
               
We've defined three local atoms, which will store our `Hammer.Manager` as well as the current state of the user's interaction (that we've called a `!zoom`).  We use that state to apply a transform style to the image, using a function called `transform`:
        
{% highlight clojure %}
(defn transform
  "Generates a cross-browser style for performing efficient CSS transforms"
  [{:keys [x y scale]}]
  (let [xform (str "translate3d(" x "px, " y "px, 0px) scale(" scale "," scale ")")]
    {:WebkitTransform xform
     :MozTransform    xform
     :transform       xform}))
{% endhighlight %}  

By using `translate3d` instead of playing around with the CSS `top` and `left` properties, we give the browser the chance to hand off the transformations we'll be applying to a GPU, giving us a significant performance boost.  There are three properties we're interested in: the x and y pan positions, and a scaling multiplier.  


## Hammer time

Now we've got a component that displays an image.  The `Hammer.Manager` needs to bind to a DOM node, so we have to create it after React has mounted our image component.  We can add a `:component-did-mount` lifecycle handler, which gets a reference to itself passed in by Reagent.  

_Note: you'll notice we've used the [`js-invoke`](http://cljs.github.io/api/cljs.core/#js-invoke) function to call the manager's functions - the stubs in the `[cljsjs/hammer "2.0.4-5"]` jar don't work, so we call the manager directly._

Here's our `:component-did-mount` lifecycle fn:

{% highlight clojure %}
:component-did-mount
(fn [this]
  (let [mc (new js/Hammer.Manager (r/dom-node this))]
    (reset! !hammer-manager mc)))
{% endhighlight %} 

We've created a new `Hammer.Manager` called `mc` and added to it a new `Hammer.Tap` Recogniser called `tap`.  In `tap`'s constructor we've defined a new event called `doubletap`, which we've defined to be two of the tap events that the `Hammer.Tap` Recogniser knows to look out for Finally we've stored the Manager in an atom.  Why?  So we can clean it up when the component unmounts with a `:component-will-unmount` lifecycle fn:

{% highlight clojure %}
:component-will-unmount
(fn [_]
  (when-let [mc @!hammer-manager]
    (js-invoke mc "destroy")))
{% endhighlight %} 

   
## Gesticulating

Great!  We've set up Hammer to watch for gestures.  We'd like the user to be able to zoom with doubletaps, pan the image by pressing and dragging, and also pinch-to-zoom.

### Doubletap
   
{% highlight clojure %}
(js-invoke mc "add" (new js/Hammer.Tap #js{"event" "doubletap" "taps" 2}))
(js-invoke mc "on" "doubletap" #(if (= 1 (:scale @!zoom))
                                 (swap! !zoom assoc :scale 2)
                                 (swap! !zoom assoc :scale 1)))
{% endhighlight %}                                 
   
In order for Hammer to detect gestures, we have to give it Recognisers.  Here we've created one and at the same time defined a new event called `doubletap`, which we define as being two of the built-in tap events.  When we see one, we've asked Hammer to call our handler function, which sets the current scaling multiplier to 2 - i.e. we'll double the size of the image.  If the scale has already changed, we reset it to 1 instead.

### Pan

We need to add a `Hammer.Pan` Recogniser and attach it to the Manager in the same way:

{% highlight clojure %}
(js-invoke mc "add" (new js/Hammer.Pan #js{"direction" js/Hammer.DIRECTION_ALL 
                                           "threshold" 0}))
(js-invoke mc "on" "panstart" #(reset! !start-zoom @!zoom))
(js-invoke mc "on" "pan" #(let [{:keys [x y]} @!start-zoom]
                           (swap! !zoom assoc :x (+ x (.-deltaX %))
                                              :y (+ y (.-deltaY %)))))
{% endhighlight %} 

Here we're allowing the user to pan the image in any direction (by default up and down are disabled, so as to avoid interfering with scroll), and we aren't requiring a threshold of movement to be reached before we respond to the gesture - we want panning to be precise and immediate.  

There's a hitch here though: the `translate3d` CSS transform takes an absolute value, whereas Hammer's gesture events are giving us deltas between the state at the start of the gesture and now.  That's where our `!start-zoom` atom comes in - it's a place we can record the current state each time the user starts a new gesture.  Hammer allows us to detect the start and end of each gesture as well as the gesture itself, in the case of pan these are called `panstart` and `pan` (all the events provided by each Recogniser are listed in the [docs](http://hammerjs.github.io/recognizer-pan/)).  When the user starts panning, we record the current zoom in the `!start-zoom` atom; then as they pan around, we can add the deltas from the incoming events to the values stored in there.

### Pinch

Enabling pinch-to-zoom is very similar.  It's natural to want to both pinch and pan at the same time, and we could allow this using Hammer's [`recognizeWith`](http://hammerjs.github.io/recognize-with/) support, which allows the detection of multiple gestures simultaneously; luckily for us though, Hammer realises that this is a common requirement and the `Hammer.Pinch` Recogniser already has a `pinchmove` event that does exactly what we want.  As with pan, we want to record the starting pan and scale so we can apply the delta from each event of the current gesture:

{% highlight clojure %}
(js-invoke mc "add" (new js/Hammer.Pinch))
(js-invoke mc "on" "pinchstart" #(do (reset! !start-zoom @!zoom)
                                     (.preventDefault %)))
(js-invoke mc "on" "pinchmove" #(let [{:keys [x y scale]} @!start-zoom]
                                  (reset! !zoom {:x     (+ x (.-deltaX %))
                                                 :y     (+ y (.-deltaY %))
                                                 :scale (* scale (.-scale %))})
                                  (.preventDefault %)))
{% endhighlight %} 

_Note: the `(.preventDefault %)` lines stop the `doubletap` gesture from falling through to components underneath our zoomer, and eventually to the browser - this is important for preventing browser zoom on mobile safari, which [doesn't respect `user-scalable=no`](http://stackoverflow.com/a/38573198/513684)._
        

## Touch me

You can find the source on [GitHub](https://github.com/conan/zoomer).  Here's a [running example](/zoomer).  

The final `zoomer.core` namespace looks like this:

{% highlight clojure %}
(ns zoomer.core
  (:require [cljsjs.hammer]
            [reagent.core :as r :refer [atom]]))


(defn transform
  "Generates a cross-browser style for performing efficient CSS transforms"
  [{:keys [x y scale]}]
  (let [transform (str "translate3d(" x "px, " y "px, 0px) scale(" scale "," scale ")")]
    {:WebkitTransform transform
     :MozTransform    transform
     :transform       transform}))


(defn zoomer
  []
  (let [!hammer-manager (atom nil)
        !zoom           (atom {:x 0 :y 0 :scale 1})
        !start-zoom     (atom {:x 0 :y 0 :scale 1})]
    (r/create-class
      {:component-did-mount
       (fn [this]
         (let [mc (new js/Hammer.Manager (r/dom-node this))]
           ;; Doubletap
           (js-invoke mc "add" (new js/Hammer.Tap #js{"event" "doubletap" "taps" 2}))
           (js-invoke mc "on" "doubletap" #(if (= 1 (:scale @!zoom))
                                            (swap! !zoom assoc :scale 2)
                                            (swap! !zoom assoc :scale 1)))
           ;; Pan
           (js-invoke mc "add" (new js/Hammer.Pan #js{"direction" js/Hammer.DIRECTION_ALL 
                                                      "threshold" 0}))
           (js-invoke mc "on" "panstart" #(reset! !start-zoom @!zoom))
           (js-invoke mc "on" "pan" #(let [{:keys [x y]} @!start-zoom]
                                      (swap! !zoom assoc :x (+ x (.-deltaX %))
                                                         :y (+ y (.-deltaY %)))))
           ;; Pinch
           (js-invoke mc "add" (new js/Hammer.Pinch))
           (js-invoke mc "on" "pinchstart" #(do (reset! !start-zoom @!zoom)
                                                (.preventDefault %)))
           (js-invoke mc "on" "pinchmove" #(let [{:keys [x y scale]} @!start-zoom]
                                            (reset! !zoom {:x     (+ x (.-deltaX %))
                                                           :y     (+ y (.-deltaY %))
                                                           :scale (* scale (.-scale %))})
                                            (.preventDefault %)))
           (reset! !hammer-manager mc)))

       :reagent-render
       (fn [_]
         [:div.zoomer
          [:img {:src   "/images/clojure-logo.png"
                 :style (transform @!zoom)}]])

       :component-will-unmount
       (fn [_]
         (when-let [mc @!hammer-manager]
           (js-invoke mc "destroy")))})))


(defn init! []
  (r/render [zoomer] (.getElementById js/document "app")))
{% endhighlight %} 


 