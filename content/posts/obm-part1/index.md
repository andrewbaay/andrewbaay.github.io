---
author: Andrew Caleb Baay
title: Implementing Operation Black Mesa's Renderer - Part 1 - Doing what we can on D3D9.
date: 2024-07-22
description: OBM Deep Dive
categories: ["Programming"]
tags: ["Source Engine", "Graphics Programming", "Game Engine"]
---

{{< gallery match="images1/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

of5a2.

## Introduction

Hi! I am Andrew, also known as Sears, and I was an Engine Programmer / Technical Artist at Tripmine Studios. I will be discussing and showcasing some of the new features I have been adding to the project over the past 4 years or so, as well as the struggles and benefits to implementing these features.

I have always been interested in how games do their graphics since I was a kid, and would always tinker with skin-mods, and do random stuff on reshade.

For the past 4 years, I have been working with the talented team at Tripmine Studios on Operation: Black Mesa. It is a remake for Half-Life’s Expansions, more specifically Half-Life: Opposing Force, and Half-Life: Blue Shift that runs on the Source Engine. Operation: Black Mesa is a huge endeavor, both in keeping in line with the story we are meant to recreate, and how we present that story in a more modern fashion.

This is a high-level overview of the techniques that are implemented on the D3D9 version of the game and is subsequently ported to the D3D11 version.

All techniques presented here are possible on Source's D3D9 Renderer.

## Goals

To fully realize my vision for the game’s presentation, we have to set some general goals:

1. An overhauled materialsystem shader backend that allows materials to be used anywhere regardless of what's rendered.
2. A unified, fully dynamic lighting system that can replace the standard lighting system that Source Engine typically has.
3. Optimize Source Engine’s rendering system to cope with the detail and performance normally present in games of today.

## Starting Out

Starting out on a project as large as OBM is a daunting task, and so i laid my goals for the game right from the start. and while the scope has grown in size, the end goal is the same, it was the goals listed above. To achieve this, i first learned how newer source engine versions handles materials differently from the SDK. the largest difference is the introduction of [4 Way Blends](https://developer.valvesoftware.com/wiki/Lightmapped_4WayBlend). This increased world geometry detail but is only supported on the CS:GO Engine and only for displacements. 

At first, i only ported my test assignments (Lens Flare, Chromatic Aberration, etc) to the new engine, but as i was doing that i was experimenting with how Projected Textures work on the CS:GO Engine.

{{< gallery match="images2/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

[Projected Textures](https://developer.valvesoftware.com/wiki/Env_projectedtexture) are a light type present on almost all Source Engine games that render dynamic shadows. it is typically used for the flashlight of the player. On newer games though (Portal 2 for instance), it was used on other light sources like the test chamber lights, as well as changing its filter (HL2, SDK Projected Textures used noise filter, while Portal 2, CS:GO Projected Textures used a gaussian filter.). Later versions of the entity also introduced a volumetric feature, mainly used on SFM. 

As disccused earlier on earlier posts, Projected Texture lighting on the Source Engine are rendered by rendering the models involved twice, once with the normal shading, and one more time with the specific light's shading. This didnt change with the newer versions of Projected Textures. This is the reason i quickly abandoned the idea of improving them and search for alternatives.

## Deferred?

{{< gallery match="images3/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

*OBM's G-Buffer Layout*

I joined Tripmine partly because I was inspired by [Black Mesa's](https://store.steampowered.com/app/362890/Black_Mesa/) Graphical overhauls over the original. this has inspired me to be interested on how do things get redone, how to make something look as good as it is despite the old engine used, and other things. One thing that caught my attention while playing the game is their dynamic lighting system. It was a deferred lighting system where they have a pass that renders the G-Buffer that they have, outputting material and spatial properties to a set of textures, and then using those textures on a lighting pass that uses box light proxies to render lighting. More information can be read [here.](https://chetanjags.wordpress.com/2022/06/01/blackmesa-xenengine-part1-a-heavily-modified-source-engine/)

I considered adapting the same strategy however implementing a dedicated G-Buffer pass we run unto the same problem as multipass lighting: We are rendering the scene twice. (This time once on the gbuffer, and another on the final pass). This has a notable advantage of using the lighting result of the deferred pass on a forward shader, alleviating any translucency issues if we turn on/off the deferred lighting on a per-renderable basis, however we are rendering the scene twice so I realized that is not the way to go.

Instead I implemented a basic forward + gbuffer approach and re-arranged the rendering code. I kept the basic forward rendering code, however at the very end of the shader i added the G-Buffer outputs aside from the normal rendering output:

```
... basic forward shading happens here

o.color0 = finalColor.xyzw;
o.color1 = float4(albedo.xyz, alpha);
o.color2 = float4(worldNormal.xyz, sunMask);
o.color3 = float4(roughness,metalness,ao,depth);

return o;
```

Implementing the G-Buffer pass as integrated to the forward shading has a major advantage of us not requiring a separate G-Buffer pass, but it comes as a cost of needing to re-arrange some of the rendering code to avoid rendering G-Buffer data to translucents, and applying the lighting before rendering translucents. I think at this time the cost is worth it to avoid rendering the scene twice.

One more thing that i differed in terms of implementation is that instead of using Boxes to render lighting, I used spheres / cones depending on Light Type (spheres for point lights, cones for spot lights.) This has an added vertex pressure to the GPU but it saves us pixel shader invocations on parts of the light entity that isnt really affected by the light. It is better explained by the picture below.

{{< gallery match="images4/*" sortOrder="desc" rowHeight="50" margins="5" 
thumbnailResizeOptions="1200x1200 q90 Lanczos"
resizeOptions="200x200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

The yellow diagram is a box solution, it encompasses the light, but loosely so, a triangle / frustum proxy mesh (Orange) more closely resembles how a spot light looks like and fits more, wasting less pixel shader invocations.

## Shadows

To render shadows for these lights, for spot lights i used typical projected shadowmapping. However, point lights are another beast, since point lights affect all around it, projected methods will not work/will have a lot of distortions, especially on the edges.

For point lights, i did consider [DPSM](http://gamedevelop.eu/en/tutorials/dual-paraboloid-shadow-mapping.htm) (Dual Paraboloid Shadow Mapping) but the inherent limitation by using vertex shader to do the paraboloid projection hurts low-poly surfaces, especially the world geometry.

{{< gallery match="images5/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

*OBM's Shadow Implementation. Left image is the 2 DPSM views to render each hemisphere, Right image is the lit image with DPSM artifacts*

Since DPSM proved to be an inadequate solution especially on world shadow casting geometry, I opted for cubemap shadows instead. This requires the shadowmap to be rendered on all 6 faces of the light, but suffers minimally from artifacts.

{{< gallery match="images6/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

I also experimented with alternative shadowing techniques like [Variance Shadow Mapping](http://igm.univ-mlv.fr/~biri/Enseignement/MII2/Donnees/variance_shadow_maps.pdf) and [Exponential Shadow Mapping](https://jankautz.com/publications/esm_gi08.pdf), but later decided that the simplicity of a Gaussian PCF kernel outweighs the potential performance benefits of such alternative shadowmapping methods.

All the shadows are merged together in a Shadow Map Atlas to save texture slots for D3D9 (D3D9 has a maximum of 16 texture/sampler pairs for each shader.)

{{< gallery match="images7/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

## Area Lighting

{{< gallery match="images8/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

Real-world lighting doesnt come from a single infinitesimal point, but rather, there is an object that emits the light, and that object has a shape. For OBM I implemented 2 types of shaped lights: Capsule and Planar. Spherical lights can be achieved using the capsule mode without any length, only a radius. Planar lights need side length, side width, and a light normal to determine the plane.

The light shape not only affects the specular reflections, but also the volumetric lighting that the light emits as well.

## Volumetric Lighting

{{< gallery match="images9/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

Aside from normal surfaces, in order to have a realistic unified lighting, the lighting must also affect volumetric objects. This includes translucent surfaces, Smoke, Fog, and other objects. In order to achieve this, I implemented a froxel solution on D3D9 using a texture strip atlas. 

{{< gallery match="images9-2/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

*OBM's Volumetric Texture Strip (Left) derived from scene lighting, at specific depths from the camera.*

Each section of this atlas corresponds to a specific depth from the camera in which we capture the lighting information. We store this information to the section, and then on translucent objects, we look up this pseudo 3D texture for radiance information at that specific point. This is incomplete since it lacks directional information but it works well enough for volumetrics.

To Render the lighting to the strip, we do:

```
    for slice in numSlices:
        zDepth = ((slice / slices) * farz) + nearz;
        worldPosition = depthToWorldPosition(zDepth, matViewProjInv);
        lighting = CalculateLightVolumetric(worldPosition);
```

To Render the lighting from the strip to a translucent renderable (or per raymarch step), we do:
```
        currProjPos = WorldToTexture(currWorldPos, matViewProj);   
        volLighting += tex2D3DStrip(volumetriclighting, float3(texCoord.xy, currProjPos.z), numSlices, farz);
```

For fog volumetric lighting i did a custom approach: Instead of rendering volumetric fog by raymarching from the camera to the specific depth, I instead created a fog volume entity. The idea behind this entity is that inside is a localized fog volume. This is similar to Black Mesa's Xog implementation, but this version, as stated earlier, can catch dynamic lighting.

Using the bounded volume fog idea, instead of raymarching from the eye to the surface, we can now raymarch within this volume, therefore making sure that we only raymarch where its actually needed.

## Viewmodel Shadowing

{{< youtube kNOjQCsh3ec >}}

{{< gallery match="images10/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

In addition to typical world shadowing, in order to ground the player into the Black Mesa Facility, I reimplemented viewmodel lighting to allow for per-light viewmodel shadows that affects all lights, even the old ones too.

This tech is made possible by combining 2 shadowing techniques. normal projected shadow mapping and Screen-Space Shadows.

To render the projected shadow, a custom shadowmap view is made pointing from the light position to the viewmodel position. the Origin and the FOV of the view is then adjusted depending on the Viewmodel AABB in worldspace. This results on a shadow that is crisp even if the casting light is far away. Screenspace shadows then fixes the peter panning of the projected shadow. 

## Misc Lighting Features

{{< gallery match="images11/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

{{< youtube 0ki750RBgeg >}}

In order to ground the viewmodels to the world they live in more, I implemented viewmodel inherent lighting. We attach the new lighting system (complete with shadows!) to certain attachment points of the Viewmodel. It reacts and moves depending on the firing state/animation of the viewmodel. Currently supported on the Displacer, Tau, Snark, and other guns.

Battery pickups also support this feature, creating shadows on the battery as shown above.

## Post Processing Stack

In addition to the lighting and shading overhauls, I also implemented a custom post processing stack for OBM. This consists of an `env_post_process` entity and volume that has a material for the specific post process shader that they want to render on that particular area.

{{< youtube eaFsvl3N5JA >}}

This allows for multiple (up to 4, can be configured) post process materials be rendered at the same time.

## Lens Flares

{{< gallery match="images13/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

I implemented lens flares which can be created at a point defined in space, or from the sun itself. Like volumetric lighting, lens flares are a great way to show a player how bright a light source is. From a design standpoint, lens flares can also be used to guide a player towards an exit in a very dark space, or blind a player when looking at a very bright light source.

## Chromatic Aberration

{{< gallery match="images14/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

Chromatic aberration is a type of post processing effect that scatters light to create an odd, somewhat otherworldly look. This was a must-have system considering the Xen segments players explore in both Guard Duty and Operation: Black Mesa. The thing with post-processing techniques is you need to think of them like garlic when cooking. A little is good, but too much can ruin the dish. Below you can see a screenshot of how this subtle light-altering effect enhances the scene when you approach a Xen Portal, when combined with color correction.

## Screenspace Reflections 

{{< gallery match="images15/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

To complement the cubemap specular reflection, I Implemented screenspace reflection support. Screen space reflections is a technique used to create more accurate, and in some ways a bit more detailed reflections, This works by raymarching the reflection vector of the surface normal against a depth buffer. if it hits the depth buffer, that means that the pixel color on that particular hit is the reflection. This doesnt work as well as Ray-Traced reflection, but looks good enough especially if you have cubemaps as a fallback.

## Bloom

{{< gallery match="images16/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

To properly convey extremely bright specular highlights, or light sources, I implemented a new implementation of bloom to the engine. this works by doing a loop between all the mips of a copy of the framebuffer texture and using a blur pass using the previous mip's output to the next mip. This progressive blurring allows for large bloom while keeping the small but bright pixels defined.

All of the above post process effects can be defined on a material and be used for the `env_post_process` entity.