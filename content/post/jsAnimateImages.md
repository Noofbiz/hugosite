---
title: JsAnimateImages
date: 2018-05-23T10:16:06-04:00
lastmod: 2018-05-23T10:16:06-04:00
author: Jerry Caligiure
cover: /images/jsAnimateImages/cover.jpg
categories: ["programming", "web"]
tags: ["js", "animation"]
---

Animate an html image element with a little js and a webworker!

<!--more-->

While working on my site for SkeleboyStudios, I initially wanted to create an
interactive page utilizing `engo` and embedding the game in there as the main
way to navigate and learn about the site. After doing so, it looked a bit cheesy
and we decided to go forward with a more traditional website.

However, while doing the site, I wanted to animate the pictures used for the
navbar as well, but I didn't want to make whole separate games and animate them
in `engo`. WebGL seemed like it would be pretty expensive for such a small task.
With just a little bit of javascript and a webworker, I was able to animate them
without a canvas or WebGL.

First you need an html image element. I used an ID to easily target it.

```html
<img id="navHome" src="/assets/img/Home1.png" alt="home navigation bar button" />
```

Next, you'll add the following javascript to the page.

```js
var homes = ["Home1.png", "Home2.png", "Home3.png"];
var path = '/assets/img/';

document.addEventListener("DOMContentLoaded", function(event) {
  var home = document.getElementById('navHome');
  if (typeof(Worker) !== "undefined") {
    if (typeof(w) == "undefined") {
      w = new Worker("/assets/js/navWorker.js");
      var i = 0;
      w.postMessage("");
      w.onmessage = function(e) {
        i++;
        if (i >= homes.length) {
          i = 0;
        }
        home.setAttribute('src', path + homes[i]);
        w.postMessage("");
      }
    }
  }
});
```

The first thing this does
is sets up the path for the images to use as animation frames.

Next, we make sure the page is loaded, then it gets the image element. If the
browser doesn't support webworkers it'll only show the first frame of your
animation. If webworkers are available, it sets up the worker using the following
js in a separate file, in this case /assets/js/navWorker.js

```js
onmessage = function(e) {
  setTimeout(function () {
    postMessage("");
  }, 100);
}
```

All this does is when the worker recieves a message, it waits 100ms then sends
a response back. When the response is recieved in the original thread, it changes
the image source to the next one on the list, then posts a message to the worker again.
Since the threads are sending responses to one another, this loop will never end.
However, because it's done through the webworker and the message callbacks, it
won't stall your main thread to wait for completion! You can do this with multiple
image elements, and even with other things such as webGL on the same page!

There are a couple other ways to animate using CSS3 or HTML5, but the CSS3
fade in / out stuff while stacking the images on top of one another seemed a bit
hacky to me, and I thought this was a simpler solution.
