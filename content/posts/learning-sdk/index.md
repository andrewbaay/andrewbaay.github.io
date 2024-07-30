---
author: Andrew Caleb Baay
title: How I learned Graphics Programming on the Source Engine
date: 2024-07-22
description: A brief summary of my source engine programming shenanigans
categories: ["Programming"]
tags: ["Source Engine", "Graphics Programming", "Game Engine"]
---

**This is a retrospective about how I learned graphics programming by the tried and true method of messing around and find out. This is NOT a tutorial, but rather just ramblings about old code, bad decisions, and the pandemic.**

## Why?
---
I love the Half-Life series, from the first one that revolutionized story telling, to the second which raised the bar on physics and realistic graphics. I played it countless times as a kid up to my teens. As time went by though, Half-Life 2's graphics aged and more graphically impressive games released. And then out of curiosity, i thought, how can i bring more modern rendering features present on later games like real time shadows to Half-Life 2's aging engine? And so I embarked into actually learning how games do their graphics deeper.

## Goals.
---
To actually learn how to implement graphical features on the Source Engine, I have to learn how the graphics pipeline first. I have no idea what it means when i was still learning the Source engine, and so i learned as i go. In other words, i learned backwards, setting large goals first and learning how to implement them step by step. Basically, i was what you call a graphics nerd, but knowing actually nothing about implementing them, i only have surface-level information.

Early on, i realized that what i want on this project is to just learn and experiment, hence i didnt set myself a release date.

I want to implement several things on my SDK fork, namely:

- Cascaded Shadow Mapping (or any realtime shadow solution)
- Dynamic (moving) light system, like point lights or spot lights
- Screenspace Ambient Occlusion
- Screenspace Reflections

Realizing this is a large task since the Source Engine is pretty old and the publicly available SDK doesnt have the full source code of the engine, I resorted to learning the intricacies of the available source code to hack my way into implementing such features.

## Starting Out
---
Implementing all the exciting stuff immediately seems like a daunting task, and so i started with smaller stuff first, like implementing a godrays shader.
\
\
{{< gallery match="images/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="900x900 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}
\
To implement this effect, i referenced [Source Shader Editor](https://youtu.be/82AMwQGjc8Y?si=CexoUEcOgoL1a1wb)'s version which uses a Skymask pass. this is the pass that only passes the color if the pixel is part of the skybox, otherwise it will be black. and then, after the skymask has been retrieved from then scene, we do a radial blur pass in which we set the center of the blur on the sun's location on the screen. Finally, we additively blend the result to the framebuffer.

{{< gallery match="images2/*" sortOrder="desc" rowHeight="10" margins="50" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="200x200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

As time went on, i eventually learned that the Alien Swarm SDK has a dynamic sunlight entity named [`env_global_light`](https://developer.valvesoftware.com/wiki/Env_global_light). This entity uses the HL2's flashlight system to render dynamic lighting from the sun. I tested it for a while but i realized that the performance scales poorly when there are more than 2 objects on the scene. I didnt knnow this back then but I will realize it down the line. The HL2 Flashlight system actually draws the object twice (once on normal lighting, another with the flashlight), and so you are effectively halving your rendering budget since everything is rendered twice.

{{< youtube XEo_bkt3d_c >}}

This is an ad-hoc solution to a dynamic shadow system i implemented. It uses 3 `env_global_light`s with a projected texture. Each "cascade"'s projected texture projects a white texture with a black hole on its center. the "hole" is supposed to contain the next cascade. this provides cascade transitions if the 3 cascades are overlaid on top of each other. As you can guess, this is not a great solution since you render the scene 3 times!, the cost is even more if you separately use the flashlight.

{{< gallery match="images3/*" sortOrder="desc" rowHeight="10" margins="50" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="200x200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

I also messed around with indoor lighting. despite the fact that i do not know how to write shaders (or what shaders even really mean), i know a little bit of C++ and with a little bit of tinkering, i managed to do a simple sorting of all projected textures based on distance to camera, and set a preferred resolution for each (this also depends on how many shadows at that resolution level, so if there are too many shadows near each other it will be lower res).

## Shaders?
---
It was around this time when i first learned about what shaders actually are. they are small code that we run on the GPU to render a pixel (Fragment/Pixel Shader), transform a vertex (Vertex Shader), or any general purpose computation (Compute Shaders), and i learned how to mess around with them on Source using this [tutorial](https://developer.valvesoftware.com/wiki/Shader_Authoring). (Source Engine is a DX9 Engine by default, so it only supports programmable Vertex and Pixel Shaders.) It was also around this time when the pandemic started, and so suddenly i have a lot more time to learn to realize my hobby which is to write cool graphics tech. I also learned about PBR around this time and *why* it was a big deal back then. and so i asked around and with the help of Max, i learned about how PBR works (albeit on a very high level since i dont really know a lot of stuff yet.)

{{< gallery match="images4/*" sortOrder="desc" rowHeight="10" margins="50" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="200x200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

I also learned that there are multiple types of shaders, like post process shaders which draws a fullscreen quad on top of the image, or a surface shader, which is applied with transformations usually to world space, then screen space, to render it. 

{{< gallery match="images5/*" sortOrder="desc" rowHeight="10" margins="50" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="200x200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

I personally find this pause menu effect really cool at the time, as i have an idea about how depth works at this point, and so i thought, what if i blur something depending on depth? i copy-pasted a simple bokeh blur shader from shadertoy, chuck it on my shader and then used `lerp` to fade in between the original color and the blurred color. I then put an overlay of a 2nd texture on the unblurred color + tint (orange in this case.)

## Advanced Stuff
---
Afterwards, i learned about [Biohazard's CSM implementation](https://github.com/Biohazard90/g-string_2013), which inspired me to continue my work on my own version. I didnt realize at first the technical details of how he did his CSM implementation on the engine, but i was amazed on how it works, albeit its flawed (it relies on dot of the sun color with the embedded lightmap, instead of a dedicated sun lightmask). I took inspiration from his work and made my own janky version.

{{< gallery match="images6/*" sortOrder="desc" rowHeight="10" margins="50" resizeOptions="200x200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}
{{< youtube SVWgSgLL45c >}} 
{{< youtube h4HmqVOlU34 >}} 

Unlike the older CSM implementations, this version is fully integrated to the default shaders `LightmappedGeneric`, and `VertexLitGeneric` (OH MAN do these shaders take ages to compile!, i didnt know it fully well back then, but the reason for this is that the CSM shader adds a permutation to the already bloated permutation lists of the standard Source Shaders. For more info read [this](https://therealmjp.github.io/posts/shader-permutations-part1/)).

After that i just messed around with more indoor and outdoor lighting, implementing POM with self shadowing (this is really expensive and i have no idea how to optimize it so i gave up)
{{< gallery match="images7/*" sortOrder="desc" rowHeight="10" margins="50"
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="200x200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

I evidently scope creeped my initial goals and i also learned that Bio's CSM implementation contains a volumetric implementation aswell. This volumetrics version is a mesh type one, wherein a mesh is placed in front of the light to catch its lighting ( and shadows )

{{< youtube 0nxFa-20NDE >}} 

At some point i also implemented SSAO / Directional Occlusion aswell. Albeit this is me still following tutorials blindly, i kind of understand how it works at this point.

{{< gallery match="images8/*" sortOrder="desc" rowHeight="10" margins="50"
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="200x200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

## Actually Good Source?
---
At this point the scope of this "experiment" is getting larger and larger and i have the motivation to make it a "base" off of other mods to work from. and so i started the AGS(Actually Good Source) Project. this project, as mentioned earlier, wants to be a "base" for other mods to make mods off of, with better graphics. i never knew back then that such an idea is ill-fated, but i ventured on with newfound motivation. In a way nothing really changed since i never really meant to release this project, it was meant to be just a playground for me to learn. However at this point i want to release it, especially at the time the pandemic is raging and i thought i will develop this thing (despite all the scope creep) on a reasonable timeframe.

Actually Good Source (AGS) became my main project as a testing hotbed for my new graphics ideas, anything that might be possible on the old D3D9 api i tried implementing there, including per-light volumetrics and CSM volumetrics.

I started optimizing the renderer i currently have, starting with the CSM system. One of the major features i added to the CSM system is to cache the static objects on a "static megatexture" shadowmap, such shadowmap is a huge one that usually encompasses the whole map and has a large resolution, usually 8192x8192. This texture is only updated once, or periodically (once every 5 seconds). and then there is another shadowmap that is the main "CSM" which only dynamic(moving) objects render to, per frame.

What happened is that the runtime performance for the CSM significantly increased since we are only rendering the dynamic objects per frame. the static objects are already rendered to the "megatexture" from the start, and since they do not move, no need to render them again. If the light/object moves, we need to re-render its shadow.
I called the cached CSM "Mixed lighting mode" since we are essentially mixing the dynamic shadows with the static ones.

{{< youtube OhnpWUDmkPs >}} 
{{< youtube GbK1rscU0aE >}} 

I also scope implemented Viewmodel shadowing, and basic point lights using deferred shading. 

{{< gallery match="images9/*" sortOrder="desc" rowHeight="10" margins="50" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="200x200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

{{< youtube 0Y9pyIwB_lM >}} 

I also messed around with custom water wave vertex deformation.
{{< youtube OKKUUSLSwok >}}

## Ending Thoughts
---
That covers most of my experience learning Graphics Programming with the Source Engine SDK. The old technology for sure provides a unique challenge on ways to implement newer rendering features on it. Overall, the project is a fun learning experience for me and enabled me to understand how to program in C++, use version control system (Git/SVN) properly, as well as learning how games render their stunning visuals. I now program properly instead of being the typical Graphics Ideas Guy :P

Thanks for Reading and Have Fun!
