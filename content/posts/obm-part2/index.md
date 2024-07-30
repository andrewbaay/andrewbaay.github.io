---
author: Andrew Caleb Baay
title: Implementing Operation Black Mesa's Renderer - Part 2 - Lightmaps and Stretched Models
date: 2024-07-22
description: OBM Deep Dive
categories: ["Programming"]
tags: ["Source Engine", "Graphics Programming", "Game Engine"]
math: true
---

{{< gallery match="images1/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

*of5a2 on OBM's Renderer. Featuring Directional Lightmap Specular support.*

## Introduction
---
Hi! I am Andrew, also known as Sears, and I was an Engine Programmer / Technical Artist at Tripmine Studios. I will be discussing and showcasing some of the new features I have been adding to the project over the past 4 years or so, as well as the struggles and benefits to implementing these features. 

**All performance metrics shown in this article are not indicative of final performance of the game, as the game is still WIP.**

## Directional Lightmaps
---
Half-Life 2's lightmapping system consists of 3 separate colored lightmaps corresponding to bump basis. This system is described in detail on [Valve Source Shading](https://cdn.akamai.steamstatic.com/apps/valve/2004/GDC2004_Half-Life2_Shading.pdf). This allows the bumpmapped surface to react to any directional lighting conditions, because each light's contribution to the hemisphere is averaged out on those three basis directions.

However, this only covers the direct diffuse lighting. Specular is handled by cubemaps only, resulting on a flat look on the material.

In order for the normal baked lights to support specular lighting that the new deferred lights currently support, I implemented a secondary lightmap for OBM that encodes the directional information of the brightest/most significant light that affects that surface, this, alongside the normal colored lightmaps, can complete the rendering equation by supplying the directional information of the light that affects that surface. Afterwards, we can just render the lighting using:

```
lighting += DoPBRLight(surface, LightmapColor, LightmapDirection, LightOutDirection);
```
where `LightmapDirection` is the incoming lighting direction from the light to the surface.
`LightOutDirection` is the outgoing lighting direction from the surface to the eye.

{{< gallery match="images2/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

In order to normalize specular transitions between overlapping lights, an alpha is considered to scale the specular reflection depending on how many lights is affecting the specific luxel.

{{< gallery match="images4/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}
*Comparison between the Default colored lightmaps and the directional lightmaps*

Using the normal HL2 basis lightmap system as a reference for this implementation, this system also supports up to 3 lights per surface for bumpmapped surface.

{{< gallery match="images3/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

Comparison between directional lightmapped surface vs normal Source Engine bumpmap basis lightmaps. Note the enhanced direct specular reflections on the xenian surface on the left image.

## Non-Uniform Prop Scaling
---
{{< gallery match="images5/*" sortOrder="desc" rowHeight="100" margins="5" 
thumbnailResizeOptions="1200x900 q90 Lanczos"
resizeOptions="1200x1200 q90 Lanczos" showExif="true" previewType="blur" embedPreview="true" loadJQuery="True">}}

In order to reduce modeller and environment artist workload, I added support for non-uniform prop scaling on the engine. Source, by default doesnt have any support for prop scaling. The CS:GO update for the engine introduced uniform prop scaling. This has helped environment artists to not re-export and re-import assets into the engine, however, the single scaling value doesnt support scaling along an axis, only uniformly across all axes.

In addition to implementing this visually, the static prop collision data has been updated aswell to support the new transforms.

Static prop combining will then be supported later for scaled props for optimization purposes.

## Ending Thoughts
---
Directional Lightmaps and Non-Uniform prop scaling are one of the major features I implemented that actually require full access to the engine as they require physics and map format changes. With this features, we have changed our OBM map version from the default CS:GO one and might not be easily readable by third party tools.

