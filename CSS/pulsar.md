## The Pulsar Screensaver

I miss screensavers. They're not really necessary with modern displays, but I still install XScreensaver on new computers. When I was looking for projects to practice HTML5 and CSS3 features on, I decided to try implementing some of my favorites. For now I'm limiting myself to plain old JavaScript, that is no frameworks, to make things more interesting.

I decided to start with the pulsar screensaver. It's five or six gradient planes that rotate around different axes.

### Build One Object

Filling a square with a gradient is pretty straightforward: just choose six values between 0 and 256 for the red, green, and blue, start at the top left of the square and go to the bottom right.

```
function randomGadient() { 
  var demo1 = document.getElementById("gradient1");
   var colors = new Array();
   for (var i=0; i < 6; i++) {
      colors[i] = Math.floor( Math.random() * 256 );
   }
   var style = "background-image: linear-gradient(45deg, rgb(" + colors[0] +"," + colors[1] + "," + colors[2] + ") 0%, rgb(" + colors[3] +"," + colors[4] + "," + colors[5] + ") 100%);";
   demo1.style.cssText = style;
}
```

The results are ([jsfiddle](https://jsfiddle.net/oxzaht0d/)):

![2025-10-01_10-25-39 (1)](https://github.com/user-attachments/assets/2d0ac5e5-fca3-4da9-af1b-d5a1a00ce217)

That's not what I was expecting.

I reloaded the page a few times but the random gradients looked flat and were not nearly as bold as the original. Then I noticed that the original had a color stop in the middle. That was easy enough to add, I just had to choose three more colors and stop halfway in the image.

```
function randomGradientWithStop() {
   var demo2 = document.getElementById("gradient2");
   var colors = new Array();
   for (var i=0; i < 9; i++) {
      colors[i] = Math.floor( Math.random() * 256 );
   }
   var style = "background-image: linear-gradient(45deg, rgb(" + colors[0] +"," + colors[1] + "," + colors[2] + ") 0%, rgb(" + colors[3] +"," + colors[4] + "," + colors[5] + ") 50%, rgb(" + colors[6] +"," + colors[7] + "," + colors[8] + ") 100%);";
   demo2.style.cssText = style;
}
```
([jsfiddle](https://jsfiddle.net/tp20cguL/))

![2025-10-01_10-26-24 (2)](https://github.com/user-attachments/assets/2fbd0208-07f1-4d9a-b55f-ad661b06b412)

That wasn't much better. Taking a closer look, I realized that all of the gradients in the original pulsar were the same. That would help clean up the JavaScript. One quick trip to the [Ultimate CSS Gradient Generator](https://www.colorzilla.com/gradient-editor/) and I had the CSS I needed.

```
.gradient {
   background: #ff0c0c; /* Old browsers */
   background: -moz-linear-gradient(-45deg,  #ff0c0c 26%, #30ff3e 50%, #26f23e 51%, #1e2dff 79%); /* FF3.6+ */
   background: -webkit-gradient(linear, left top, right bottom, color-stop(26%,#ff0c0c), color-stop(50%,#30ff3e), color-stop(51%,#26f23e), color-stop(79%,#1e2dff)); /* Chrome,Safari4+ */
   background: -webkit-linear-gradient(-45deg,  #ff0c0c 26%,#30ff3e 50%,#26f23e 51%,#1e2dff 79%); /* Chrome10+,Safari5.1+ */
   background: -o-linear-gradient(-45deg,  #ff0c0c 26%,#30ff3e 50%,#26f23e 51%,#1e2dff 79%); /* Opera 11.10+ */
   background: -ms-linear-gradient(-45deg,  #ff0c0c 26%,#30ff3e 50%,#26f23e 51%,#1e2dff 79%); /* IE10+ */
   background: linear-gradient(135deg,  #ff0c0c 26%,#30ff3e 50%,#26f23e 51%,#1e2dff 79%); /* W3C */
   filter: progid:DXImageTransform.Microsoft.gradient( startColorstr='#ff0c0c', endColorstr='#1e2dff',GradientType=1 ); /* IE6-9 fallback on horizontal gradient */
}
```

<img width="320" height="320" alt="2025-10-01_10-26-50" src="https://github.com/user-attachments/assets/b7dab24e-3625-40cd-a4b1-71be480b63be" />

That was not the first or last time I would make something more complicated than it needed to be.

### Do Something

Rotation is available around any axis using a CSS 3D transform. The functions rotateX, rotateY, and rotateZ are values of the transform and -webkit-transform CSS properties and can rotate an HTML object around the X, Y, or Z axis. The rotation functions each take values in degrees and can go beyond 360--- something which will come in handy later.

The rotation can be animated by applying a CSS transition to the transform. To make things simple, the transition property can be applied to all CSS properties.

The final CSS:
```
   transition: all 4s;
   -webkit-transition: all 4s;
```

The final JavaScript:

```
var rotateDemoX = 0;
function rotateX(id) {
   var rotate = document.getElementById(id);
   rotateDemoX = rotateDemoX + 180;
   rotate.style.cssText += ";transform: rotateX(" + rotateDemoX + "deg);-webkit-transform: rotateX(" + rotateDemoX  + "deg);";
}
```
([jsfiddle](https://jsfiddle.net/vnLdos8p/))

![2025-10-01_10-34-16 (1)](https://github.com/user-attachments/assets/f4649e41-0efb-4885-817b-b27d7d60433c)

### Repeat

At first I dynamically created six divs to use as my rotating planes, then I realized that I was making things more complicated than I needed to--- again. I ended up writing the planes in HTML and using assigning a negative margin-top to each so they'd end up on top of each other.

```
   <div id="p0" class="gradient"></div>
   <div id="p1" class="gradient" style="margin-top: -70%"></div>
   <div id="p2" class="gradient" style="margin-top: -70%"></div>
   <div id="p3" class="gradient" style="margin-top: -70%"></div>
   <div id="p4" class="gradient" style="margin-top: -70%"></div>
   <div id="p5" class="gradient" style="margin-top: -70%"></div>
```

The `rotateX`, `rotateY`, and `rotateZ` JavaScript functions above don't work well when applied simultaneously; the cssText property gets overwritten. That's easy enough to fix though with the rotateAll function below.

Note: the `xCol`, `yCol` and `zCol` are just constants to increase readability.

```
function rotateAll(p) {
   var rotate = document.getElementById('p' + p)
   rotate.style.cssText += ";transform: rotateX(" + planes[p][xCol] + "deg) 
                                        rotateY(" + planes[p][yCol] + "deg) 
                                        rotateZ(" + planes[p][zCol] + "deg);
                     -webkit-transform: rotateX(" + planes[p][xCol] + "deg) 
                                        rotateY(" + planes[p][yCol] + "deg) 
                                        rotateZ(" + planes[p][zCol] + "deg);";
}
```

Defining a random rotation then is fairly straightforward.

```
function rotatePlane(p) {
   planes[p][xCol] +=  Math.floor( Math.random() * 720 );
   planes[p][yCol] +=  Math.floor( Math.random() * 720 );
   planes[p][zCol] +=  Math.floor( Math.random() * 720 );
   rotateAll(p);
   setTimeout('rotatePlane(' + p + ');', 4000);
}
```

The last line allows for a continuous rotation. Since the transitions have a fixed duration, that is all the delay that is needed before the function recurses.

A setup function called on page load completes the screensaver.

```
function setupDemo3() {
   planes = new Array();
   for (i = 0; i < numPlanes; i++) {
      planes[i] = new Array();
      planes[i][xCol] = 0;
      planes[i][yCol] = 0;
      planes[i][zCol] = 0;
      rotatePlane(i);
   }
}
```
([jsfiddle](https://jsfiddle.net/mxtLr6gq/))

![2025-10-01_10-35-16 (1)](https://github.com/user-attachments/assets/32ac1477-e354-4722-99ee-044df0c95bfd)


## What Did I Learn?

* Random gradients don't work.
* Between writing the demo and publishing it, Chrome updated to use a newer version of the CSS 3 standard.Originally the rotation functions could be applied in series, that is I could rotateX(10deg) and then rotateX(10deg) and have the result be a 20 degree rotation. That doesn't work anymore. The fix was to a running total of the rotation along all of the axes. That kept the planes spinning.
