---
layout: post
title:  "Automating Linkage or: How I Learned To Stop Worrying And Love The Build"
date:   2014-08-18 16:00:00
redirect_from: /bower/gulp/wiredep/javascript/2014/08/18/automating_linkage-or-how-i-learned-to-stop-worrying-and-love-the-build.html
categories: bower gulp wiredep javascript
comments: true
---
# Building with Gulp.js

I've recently come to Javascript, and the first thing I wanted to do was set up a build for my project.  We'd been 
using [Grunt](http://gruntjs.com/), but that all looked a bit [ANT](https://ant.apache.org/)sy, so when a colleague 
recommended [gulp.js](http://gulpjs.com/), I jumped right in.

Getting started was easy, with [npm](https://www.npmjs.org/) and [Bower](http://bower.io/) downloading and installing 
the dependencies I needed; it all felt very familiar to Maven, Gradle and Leiningen I'm used to from back-end dev. But 
there's one part missing: Bower pulls the libraries, but doesn't provide them to my app.  I have to maintain my 
dependency list in my build file as well, to ensure the libraries are linked in my index.html in the right order.  

"That's not very [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself)", I thought, "I should automate this".  

I took a look around and couldn't find an existing tool to do it, so I put it together myself.  Here's how to do it.

# Dependency management with wiredep and gulp-inject

Bower knows about transitive dependencies, but doesn't expose that information in a way that can be easily consumed in 
Javascript.  Instead, we need to calculate the dependency tree ourselves by walking the `bower.json` files of each 
dependency in order.  This job is done for us by [wiredep](https://github.com/taptapship/wiredep).  Its gulp plugin 
isn't very configurable, but that's ok, because we [don't need gulp plugins](http://blog.overzealous.com/post/74121048393/why-you-shouldnt-create-a-gulp-plugin-or-how-to-stop).  Wiredep itself 
can produce a list of our dependencies and wire it into our `index.html`.  Wiredep won't do the same for our own code 
though, for that we'll use [gulp-inject](https://www.npmjs.org/package/gulp-inject).

First, we add placeholders for our CSS links and script tags to our `index.html`, which is where wiredep 
(using bower:blah) and gulp-inject (using inject:blah) will write in our tags:


{% highlight html %}

    <head>
      ...
      <!-- bower:css -->
      <!-- endbower -->
    
      <!-- inject:css -->
      <!-- endinject -->
    </head>
    <body>
      ...
      <!-- bower:js -->
      <!-- endbower -->
    
      <!-- inject:js -->
      <!-- endinject -->
    </body>

{% endhighlight %}


In the spirit of automating ALL THE THINGS, in our gulpfile let's use the 
[gulp-load-plugins](https://www.npmjs.org/package/gulp-load-plugins) plugin:


{% highlight javascript %}

    var wiredep = require('wiredep');
    var plugins = require('gulp-load-plugins')();

{% endhighlight %}


That exposes all the plugins we have in our `package.json` inside the plugins variable, and we don't have to worry about 
adding each one manually.  As we aren't using the wiredep plugin, we still require it directly.

Wiredep calculates the dependency list and stores it in its `js` and `css` variables; we can use these in our gulp 
tasks to copy the files to the `build` folder:


{% highlight javascript %}

    gulp.task('vendor-scripts', ['install'], function() {
    
      return gulp.src(wiredep().js)
    
        .pipe(gulp.dest('build/vendor'));
    
    });
    
    gulp.task('vendor-css', ['install'], function() {
    
      return gulp.src(wiredep().css)
    
        .pipe(gulp.dest('build/vendor'));
    
    });

{% endhighlight %}


Now we process the `index.html` to reference all these files.  We pipe it through wiredep and gulp-inject, and write it 
out at the end:


{% highlight javascript %}

    gulp.task('index', ['scripts', 'css', 'vendor-scripts', 'vendor-css'], function() {
    
      return gulp.src('src/index.html')
        .pipe(wiredep.stream({
          fileTypes: {
            html: {
              replace: {
                js: function(filePath) {
                  return '<script src="' + 'vendor/' + filePath.split('/').pop() + '"></script>';
                },
                css: function(filePath) {
                  return '<link rel="stylesheet" href="' + 'vendor/' + filePath.split('/').pop() + '"/>';
                }
              }
            }
          }
        }))
    
        .pipe(plugins.inject(
          gulp.src(['build/src/**/*.js'], { read: false }), {
            addRootSlash: false,
            transform: function(filePath, file, i, length) {
              return '<script src="' + filePath.replace('build/', '') + '"></script>';
            }
          }))
    
        .pipe(plugins.inject(
          gulp.src(['build/assets/**/*.css'], { read: false }), {
            addRootSlash: false,
            transform: function(filePath, file, i, length) {
              return '<link rel="stylesheet" href="' + filePath.replace('build/', '') + '"/>';
            }
          }))
    
        .pipe(gulp.dest('build'));
    });

{% endhighlight %}

    
There are two main things going on here: wiredep's injection of third party dependencies, and gulp-inject's, um, 
injection of our own dependencies into the `index.html`.

We're using wiredep's `stream()` function, so we can use it as part of a 
[node stream](https://github.com/substack/stream-handbook); it reads our project's `bower.json`, calculates the 
dependency tree, generates the `<script>` and `<link>` tags for all our dependencies, in the correct order, and injects 
them into the `index.html`.  The configuration here is telling it that the tags it creates should be pointing into the 
`vendor` directory, where we've stuck all our third party libraries in the `vendor-scripts` and `vendor-css` gulp tasks 
(above).

We also want to inject our own script files after all the dependencies, but wiredep doesn't stretch that far, so we add 
a bit of work for gulp-inject to do: we simply give it a glob that picks up our code in the output folder, and 
transforms the paths so they're relative to the build `index.html` instead of the source one.  Much like we did with 
wiredep, we tell it how to generate the `<script>` and `<link>` tags in the right way.  

# Overriding Bower

It's nearly there!  Every time I add a dependency to my `bower.json`, it'll automatically get wired into my 
`index.html`, so long as the main entry in its `bower.json` is correct.  Sadly quite a few libraries don't provide this 
information.  In these cases, you need to add an override to your `bower.json` for the problematic library:


{% highlight json %}

    "overrides": {
        "angular-gridster": {
          "main": [
            "src/angular-gridster.js",
            "dist/angular-gridster.min.css"
          ]
        }
      }

{% endhighlight %}


Bower and npm seem pretty standard now in the Javascript world, so if you find a library where the main element is 
missing, why not throw over a Pull Request to fix it?  It's likely to be quickly accepted (like the authors of the 
excellent [angular-gridster](https://github.com/ManifestWebDesign/angular-gridster) project from the example above 
[did](https://github.com/ManifestWebDesign/angular-gridster/pull/87)), removes config from your application and helps 
others.  Literally everybody wins!
