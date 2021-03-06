---
layout: post
title:  "Visual static noise with the HTML5 Canvas"
date:   2016-07-25 13:00:00 +0000
---

Everybody loves [Canvas](http://www.w3schools.com/html/html5_canvas.asp). It was probably the first Web API that actually enabled Web devs to be creative, and I'd argue it actually transformed the Internet (for the better). It's got most of the tools a creative dev needs – 2D primitives, image rendering, compositing (!!!) and with `requestAnimationFrame` even optimised animation abilities.

For my personal portfolio website, I wanted to do something that would really stick out as a technical challenge, but also create a particular _genius loci_ for the relatively simple site. Inspired by some other website that I can't remember, I decided to make a semi-transparent "wall of noise" over these lush background photos. It makes text more readable, and together with the subtle Web Audio noise in the background, really provides a feeling of presence.

Now, you might say this could easily be done with loads of gifs. But then you have to _load_ a gif, _position_ it on the screen repeatedly, and you still end up with a fixed sequence of frames. With Canvas, we can do this more efficiently and keep the ability to change the algorithm later.

Here's how I did it. You first need the following HTML somewhere in your page:

{% highlight html %}
<!-- style us as you like-->
<canvas id="grain"></canvas>
<canvas id="grain-block" style="display: none;"></canvas>

{% endhighlight %}

And now for the actual algorithm:

{% highlight javascript %}

/*
	STATIC NOISE ANIMATION GENERATOR
	by Jakub Fiala
	fiala.uk
	Licensed under the Don't Be a Dick License
	http://www.dbad-license.org

	This is cool because it's very optimised. Using requestAnimationFrame, we only calculate new noise
	when the browser needs us to.

	Also, we use a hidden utility canvas to first calculate a small chunk
	of noise, and then copy it around the main canvas with drawImage. This turns out to be a lot faster than just generating the image data and calling putImageData many times.

	Lastly, we set a "pulse" value in Hz, which determines how many times per second we'll call requestAnimationFrame.
	This helps because rAF is usually called way too often, so we can relieve the browser of rendering too much stuff.

	grainSize++ || divisions++ || pulse++ === faster performance === shittier looks

	configure with care.

	DEPENDENCIES: none, vanilla js baby.
*/


// set the globals
var grainSize = 4 //in pixels
var divisions = 4
var brightness = 0.43 //fraction of 1
var pulse = 25 //in Hz
var mainCanvasId = "grain"
var utilityCanvasId = "grain-block"

//create the main canvas context, and a utility "block" context for writing in the pixels
var canvas = document.getElementById(mainCanvasId);
var c = canvas.getContext('2d');
var gb = document.getElementById(utilityCanvasId);
gb.style.display = "none";
var gbc = gb.getContext('2d');

//set context dims
canvas.height = window.innerHeight;
canvas.width = window.innerWidth;
gb.width = window.innerWidth/divisions;
gb.height = window.innerHeight/divisions;

//initialize what we need: brightness scaled to UInt8, pixel container, and the increment for positioning blocks
brightness = brightness*255;
var imageData = c.createImageData(window.innerWidth/divisions, window.innerHeight/divisions);
var increment = 1/divisions;

//this is the main loop
var renderGrain = function() {
	//create random pixel data, scaled by brightness value
	for (var i = 0; i < imageData.data.length; i += grainSize) {
		var n = Math.round(Math.random()*brightness);
		for (var j = 0; j < grainSize; j++) {
			imageData.data[i+j] = n;
		}
	}

	//clear the main context
	c.clearRect(0, 0, canvas.width, canvas.height);

	//put pixels on the utility context
	gbc.putImageData(imageData,0,0)

	//draw on the main context
	//drawImage is much faster than putImageData
	for(var x = 0; x < 1; x += increment)
		for (var y = 0; y < 1; y += increment)
			c.drawImage(gb, window.innerWidth*x, window.innerHeight*y);

	//wait for 1s / [pulse]Hz, to give the renderer a break
	setTimeout(function(){ requestAnimationFrame(renderGrain); }, 1000/pulse);
};

window.onresize = function() {
	//recalculate dimensions
	grainSize = Math.floor(grainSize*window.devicePixelRatio)
	imageData = c.createImageData(window.innerWidth/divisions, window.innerHeight/divisions);
	canvas.height = window.innerHeight;
	canvas.width = window.innerWidth;
	gb.width = window.innerWidth/divisions;
	gb.height = window.innerHeight/divisions;
}


{% endhighlight %}
