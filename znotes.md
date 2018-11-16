Additional notes by Zhahai on how this works; read author's note first

The codes uses the canvas & brush model

# The Canvas

The canvas has one entry per output LED.  Ideally, it would have a current HSV value for the LED, and a "next value" (HSV) towards which
the current HSV value should be faded over time, moving at a given fade speed and direction.

To save space, it only has one slot for the "next value", containing either the next H, next S or next V; the other two components 
would not be faded.  The canvas element contains flags for which of the HSV components to fade and in which direction.  (No fading
is of course an option)

For Saturation and Value, fading in mean going from 0 (black or unsaturated) towards the "next value", while fading out moves in 
the other direction - towards black or unsaturated.  If both flags are set, then we will fade in towards the "next value", 
clear the fade in flag when we get to the target, and then fade out, clearing the fade out flag when we hit 0.

For Hue, we either increment or decrement the hue by one until we reach the target, circling around.  In this case there's no
combination, fading in + fading out in sequence.

The fading method is a little convoluted from avoiding floating point.  
fade speed is scaled by 10, such that a fade speed of 10 = fade one (H,S, or V unit) per cycle.
for fade speed = 20, fade two units per cycle
for fade speed = 5, face one unit per two cycles.  That is, make the change interval 2 system cycles, and the value increment 1
There is more detailed fade speed logic which may make it smoother (eg: to distinguish between fade speed 4 and 5), which could 
be deciphered if need be.

So the canvas is a 1D surface which implements fading for us, with each LED having it's own rate of fade and destination HSV
(tho the target HSV is the same as the current HSV with a new value for just one of the three components).  

The fade in followed by fade out is an additional feature.

If you try to fade more than one component, things will get a little bit wacky (fading more than one component towards the same value), 
but not break.

# Drawing to the Canvas, using the Brush

Often a brush in this model contains color, brush shape, opacity etc.  Here it contains a color, some parameters related to fading, and
also a location and speed.  Every time you draw to the canvas with the brush (one function call), a canvas element will be updated with
color and fading parameters, and the brush will be moved along the LED array, depending on its speed.  So this is an animated brush.

The idea is that you configure brush, and the let it go - calling on it every cycle to draw to the canvas and move itself.  The caller
could of course change the brush over time as well, eg: chaning it's color.

The brush has a fine location (element "i") and a course location (element "position").  The course location is the integer LED indeex 
along the string, incrementing by one goes to the next LED.  The fine location has 4096 (1 << 12) subpixel steps per LED.  That is,
we shift the fine location (i) right by 12 to get the coarse location, and then write to that LED.  Thus which location in the
canvas is being drawn to by the brush can change quickly or slowly.  For efficiency, the actual copying of color and fade parameters
to the canvas is only done automatically when the coarse location changes.

The brush can be configured to wrap or bounce on the ends; it continues moving as long as you keep calling draw().

Drawing with a brush with no fading configured just sets the destination LED to the brush's HSV color.  If one of the components is
to be faded, then drawing does not change the current value for that component (in the target LED's entry in the canvas array), 
instead copying the brush's value to the canvas element's "next value" to be faded to.  Flags are set in the canvas element in
regard to the component (if any) to fade, and the fade speed and direction.  The LED (canvas element) can optionally fade that
component to 0 afterward at the same speed (for SV), or can fade clockwise or counter clockwise on the color wheel (for H).

There is a small difference in the configuration of the brush and the retained params in the canvas in terms of hue direection; the
canvas is configured to fade hue cw or ccw, but when setting up the brush one can specify near or far path along the hue circle.
(look at this again in more detail)

# Overview

Let's ignore that only one of the HSV components can be faded at a time (for a given LED); that's just a memory saving devise.

The interesting thing about this approach is that every LED has a current HSV value and a "next value" to which it will automatically
fade with no further brush draws needed); each LED also has its own fade speed; and for S & V components has the possibility of fading
to 0 after the destination is reached; and for H component can go either direction around the color wheel.

So fading is automated in the Canvas (via action on each transfer() call), and brush movement is automated in the Brush (via updating
the location on each draw() call).  If the outer program just sets up the brush, and then calls draw() on the Brush and transfer()
on the Canvas - an animation will ensue.  Other differences can be done by changing the parameters of the brush (color, location, 
fading params) between calls.

The basic concept here is of moving single-LED brushes, which leave a trail of self fading LEDs - but each brush can leave 
a different type of fading behind. Like different comets or fireworks, each with different fade dynamics.

The Canvas is retained between cycles, unless manually cleared.

I think multiple Brushes can be animated this way; perhaps multiple Canvases.  I think multiple brushes are pretty straightforward.
I believe that each Brush would each overwrite a given pixel if there were multiple brushes passing the same pixel, last wins.

The Canvas.transfer() method adds the converted HSV -> RGB values to the existing FastLED pixel array elements, 
after clearing the latter each cycle.  More than one Canvas could be transferred after clearing the FastLED array after one
clearing.  This uses wrap-around addition, but the NeoPixel version (same author) seems to do manually implemented saturated
addition.

