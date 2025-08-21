---
layout: article
title: Campanion
mode: immersive
header:
    theme: dark
article_header:
    type: overlay
    theme: dark
    background_color: '#a97'
    background_image:
        gradient: 'linear-gradient(135deg, #a974, #110a)'
        src: /assets/images/campanion/preview_banner.gif
show_title: true
permalink: /campanion/
---
<link rel="stylesheet" href="/assets/css/index.css">

[<img src="/assets/images/logos/draft-selection-rookie-awards.webp" class="image image--xs">](https://www.therookies.co/entries/42422){:title="Draft Selection at The Rookie Awards 2025 Game of The Year (Console & PC)"} [<img src="/assets/images/logos/finalist-rookie-awards.webp" class="image image--xs">](https://www.therookies.co/entries/42422){:title="Finalist at The Rookie Awards 2025 Game of The Year (Console & PC)"}
{:.center}

<p class="gameinfo">
<b>Duration:</b> 3 months (April - July 2025)
<br>
<b>Team size:</b> 2 programmers, 3 artists
<br>
<b>Engine:</b> Unreal Engine 5
<br>
<b>My role:</b> Programming (C++, Blueprint), Shaders
</p>

*Campanion* is the final game project I worked on during my first year at Yrgo. We were tasked with forming a group of 5-8 people to create a game in 8 weeks, using either Unreal Engine or Unity - this time we chose **Unreal Engine**.

We received **tons of positive feedback** on *Campanion*, and we all continued working during the summer to polish the game before submitting it to SGA and The Rookie Awards, the latter in which it recently became **nominated as finalist** for Game of the Year out of over 250 submissions across the world.

You can check out our (now slightly outdated) game trailer below:


## Trailer

<div>{%- include extensions/youtube.html id='YBdd-pXmyXg' -%}</div>

# My work

Due to us only being **two programmers**, our work has had huge areas of overlap, but in general **I've focused more on the structural backend systems** behind all important features, rather than the gameplay implementations themselves (though I've contributed a lot in that domain too).<br>Additionally, I've spent a good chunk of time designing and implementing **many of the shaders** in the game.

Here are some of the main features I've created:

## Photograph system w/ filters

<div class="center">
<img src="/assets/images/campanion/smallchair.gif" class="image image--xl" title="Taking a photograph of a chair without any filters.">

<img src="/assets/images/campanion/thermaldark.gif" class="image image--xl" title="Taking a thermal photo of the dark to show the way forward.">
</div>

**My perhaps most important task** of this project was designing the core system that allows the player to capture an image of the environment, then save and render the resulting image onto a physical polaroid in the world.

The tricky part was getting this to **work with all the different filters** - we wanted the ability to *conditionally highlight and add effects* to specific objects in certain filters, including effects like glow or transparency, so a simple screen space shader would not cut it.

We also needed the system to be as **friendly and easy to use as possible** for our artists to use when they set up the environments, allowing them to highlight objects in filters and style environments without too much hassle.

### The requirements
I figured the mechanic could be divided into 3 main tasks:
- Taking snapshots of the scene from a specific position
- Rendering the images to a physical photograph
- Applying filters

The first two steps are relatively simple, as Unreal has a built in (albeit poorly documented) *SceneCaptureComponent2D* that can be used to **render a snapshot of the scene into a RenderTarget2D asset**. These can be created at runtime at any resolution, so it's as simple as creating a new render target 2D when you take a photograph, then spawn in a polaroid object and assign the render target as its base color.

Creating a new image asset for each photograph *may sound like a source of memory/VRAM issues*, but we actually tested this; with the current photo resolution, thousands can be created and rendered at once before issues arise.
{:.warning}

### Filters

The filters are the interesting part - the SceneCaptureComponent2D renders images completely separate from the main rendering loop. This means that **if I changed the material of an object, told the scene capture component to render, then immediately changed the material back**, this could be done in between two frames, and the player would not notice any change happening at all. The rendered texture, however, shows the object with the different material.

This is the core principle of the entire filter photo system. By telling each object in view that a photo is going to be taken with a specific filter, **the object can change its material to the appropriate one for that filter**, and immediately change it back once the image is complete. With clever uses of this, it can create really cool effects.

The architecture for this happens to be quite handy, too.

By defining a blueprint interface in C++, I can create a standardized contract for receiving the photo events from the camera. The camera then calls these interface functions on each object when a photo is about to be taken, and the object acts accordingly, depending on implementation.

<div class="center">
<img src="/assets/images/campanion/interfaceexample.png" title="Demonstration of the Invisibility shader. Hidden objects exist around the map and can only be seen through this filter.">
<p class="imagedesc"><i>Defining an interface in C++ rather than Blueprint allows me to access it in other C++ scripts.</i></p>
</div>

By then overriding the Actor and StaticMeshActor classes in C++ with my own subclass, which contains a default implementation for these interface functions, they can all **switch to the appropriate material when a photograph is taken, by default**. This eliminates the need to create a separate blueprint for each filterable mesh in the game.

<div class="center">
<img src="/assets/images/campanion/materialsettings.png" title="Mesh actors by default inherit from the custom classes that implement material switching for each filter.">
<p class="imagedesc"><i>All static meshes inherit from FilterableStaticMeshActor by default using engine redirects, opening up settings for materials for each filter for each mesh, unless explicitly specified otherwise. Custom blueprints can inherit from Filter Actor to achieve similar functionality for all child meshes.</i></p>
</div>

This also means that any object can **manually override the interface functions**, and replace the default material switching behavior with something tailored to that object.

<div class="center">
<img src="/assets/images/campanion/decalpreview.png" title="Mesh actors by default inherit from the custom classes that implement material switching for each filter.">
<p class="imagedesc"><i>Some decals override the default material switching behavior and instead entirely toggle visibility depending on the filter.</i></p>
</div>

95% of objects in the game only need to switch their material to achieve its desired effect, such as becoming transparent or highlighted in X-ray, or glow in thermal photos. But **complex objects have the freedom to override this default behavior** and do something else when a photo is about to be taken, or do nothing at all. The flexibility of this system has been vital for certain puzzles, while also letting the default behavior rule over the ordinary cases.

Simply taking an UV photo can dim the lights in a room, enable fingerprint decals, change writing on blackboards, objects can appear or disappear... **any change or any behavior is possible**, and is incredibly easy to implement per-object to then be used and tweaked by level designers.

{::comment}
**Check out my design process for this system here:**<br>
{:.center}

[READ MORE](/campanion/photos/){:.button.button--outline-success.button--rounded.button--xl}
{:.center}

(It's an interesting read, I promise)
{:.center}
{:/comment}

## The Naval Mine sequence

<div class="center">
<img src="/assets/images/campanion/bombshowcase.gif" title="The bomb-bot puzzle sequence. Not included is the sequence where you find out how to open the wire lid, and the segment where you must push down the correct pins on the bomb's body.">
</div>

As for some more *gameplay-centered* work, I also created all functionality for the naval mine puzzle - the complex set piece of a puzzle that acts as the **culmination of all the mechanics** the player has learned thus far, combined into one "boss" puzzle.

My work here included **implementing the couple animations that the artists provided**, and **creating the rest of them using code** (pressing down the pins of the robot, cutting wires, opening lids, spinning fans etc.). I did this using Unreal's animation blueprint system, which allowed me to graphically play animation montages and interpolate bones based on various logical conditions. By **abstracting the animation code away from the gameplay code**, all I needed to do was **set the appropriate flags** on the animation blueprint via the gameplay script, and the animation state updated automatically. Neat stuff.

<div class="center">
<img src="/assets/images/campanion/countdownshader.gif" title="Bomb countdown shader made in Unreal Engine's Material Editor.">
<p class="imagedesc"><i>Unused face shader that counts down from 30 when you cut a wrong wire.</i></p>
</div>

I also created the whole gameplay sequence itself, including all the interactions with the model such as pressing down handles, cutting wires, attaching RAM sticks to the rig, changing facial expressions and so on.

<div class="center">
<img src="/assets/images/campanion/bombblueprint.png" title="How the bomb looks inside Unreal's Blueprint Editor. The entire puzzle is conveniently contained inside a single Blueprint.">
<p class="imagedesc"><i>The bomb in the Blueprint Viewport, with all its interactable colliders.</i></p>
</div>

The bomb has lots of interactions with the filters in the game. Players must use the thermal filter to find which (cooling) pins are hot and should be pressed down, and the scripts must enable these thermal decals depending on the bomb state. The UV filter reveals fingerprints on the lid for opening the wire hatch. Thermal shows which wires are active and should be cut. X-ray reveals the RAM sticks inside the bomb which must be repaired.

<div class="center">
<img src="/assets/images/campanion/thermaldecalsnippet.png" title="Bomb countdown shader made in Unreal Engine's Material Editor">
<p class="imagedesc"><i>Simple blueprint snippet for enabling the pins' thermal decals based on their pressed status.</i></p>
</div>

It's a really interesting puzzle; all parts fit in so naturally with the filter photo mechanics that the player has learned to use, and it all combines into a great "boss" puzzle.

## Shaders & misc.

### Invisibility shader

<div class="center">
<img src="/assets/images/campanion/invischair.gif" title="Demonstration of the Invisibility shader. Hidden objects exist around the map and can only be seen through this filter.">
</div>

This shader was really fun to make. We wanted a filter that reveals objects that otherwise do not exist - ghost objects, if you will. After brainstorming for a while on how this can be represented ingame (some kind of blue distorted ghost shape in a black-and-white photo?), I tried a bunch of options but none really stuck.

Then I realized - specialized shaders can be applied to the photograph itself, too. Why not make something really freaky where it **cuts out the object from the photo itself**? What better way could there possibly be to visually represent an object that *does not exist*, than making its absence itself be the effect?

I achieved the result above through using 3 different shaders - one on the to-be invisible object, one screen shader, and one shader on the photograph itself.

- The object shader simply makes the entire object a solid, unlit light blue color when taking the photo. The object then gets assigned a special custom depth stencil value, so we can identify it in the screen shader.

- The screen shader then makes the entire screen monochrome, except for the areas marked with our custom depth stencil value. This gives an image where the invisible objects are highlighted in solid light blue color.

- The photograph shader then simply masks out all parts of the image that have that specific cyan color.
  - A side effect of this is that the pixels on the edges are not 100% cut out due to blurring and other effects. This is a good thing, as it gives all objects a very subtle glowing white/blueish outline that highlights it.

<div class="center">
<img src="/assets/images/campanion/invisstatue.png" title="Demonstration of the Invisibility shader. Hidden objects exist around the map and can only be seen through this filter.">
<p class="imagedesc"><i>Invisibility shader. Note how the statue casts a shadow on the floor only inside the photo.</i></p>
</div>

### Other screenshots of my work

<div class="left">
<img src="/assets/images/campanion/burnshader.png" title="Burning painting/paper shader that is displayed when a laser pointer burns down a painting" class="image image-xl">
<p class="imagedesc center"><i>Burning painting shader</i></p>
</div>
<div class="right">
<img src="/assets/images/campanion/knockables.png" title="Burning painting/paper shader that is displayed when a laser pointer burns down a painting" class="image image-xl">
<p class="imagedesc center"><i>Customizable knockable props</i></p>
</div>
<div class="center">
<img src="/assets/images/campanion/keypad.png" title="Burning painting/paper shader that is displayed when a laser pointer burns down a painting" class="image image-xl">
<p class="imagedesc center"><i>Keypad with animated buttons</i></p>
</div>
<div class="center">
<img src="/assets/images/campanion/xrayshader.png" title="Demonstration of the Invisibility shader. Hidden objects exist around the map and can only be seen through this filter.">
<p class="imagedesc"><i>The safecracking puzzle. The player can use X-ray photos to reveal the mechanism inside the safe. Each ring's pin latches onto the one behind it, you rotate the front ring back and forth until all of them align, just like a real safe.</i></p>
</div>

<hr>

**Download Campanion:**
{:.center}
[<img src="/assets/images/logos/itch/itchio-logo-white.svg" class="gamelinkbuttonlogo">](https://yrgo-game-creator.itch.io/campanion){:target="_blank" title="Download Campanion on itch.io" .button.button--outline-error.button--pill.gamelinkbutton}
{:.center}


<br>

[Back to the main page](/){:.button.button--outline-success.button--rounded.button--xl}
{:.center}

<br>
