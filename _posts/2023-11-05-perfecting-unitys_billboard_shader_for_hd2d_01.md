---
layout: post
author: thomas_walker
title:  "Perfecting Unity’s Billboard Shader for HD2D - Part 1"
date:   2023-11-06 07:45:00 +0000
categories: update
---

## HD2D

![Xenogears-(by-The-Dark-Id)/Update%2002](/assets/img/2023-11-05-perfecting-unitys_billboard_shader_for_hd2d/25-forehead33.jpg)
[Xenogears by The Dark Id](https://lparchive.org/Xenogears-(by-The-Dark-Id)/).

If you grew up with a PS1 or a DS, I’m almost certain that some of your favourite games used 2D characters in a 3D environment. Not only did it give you control of the camera to look for hidden items, it also allowed for much more expressive characters than 3D models could achieve at standard definition.

![Grandia/Update%2028](/assets/img/2023-11-05-perfecting-unitys_billboard_shader_for_hd2d/110-067.jpg)
[Grandia by Edward_Tohr](https://lparchive.org/Grandia/).

Since then, the art style saw very little use until the release of Octopath Traveller in 2018, which Square Enix described as HD2D. This combined authentic 16/32-bit artwork with modern rendering techniques such as normal maps, dynamic shadows and post-processing.

![Octopath-Traveller/Update%2004](/assets/img/2023-11-05-perfecting-unitys_billboard_shader_for_hd2d/12-013.jpg)
[Octopath Traveller by Mega64](https://lparchive.org/Octopath-Traveller/).

One of my first experiments with game engines was a 3D recreation of Aliahan, the first town of Dragon Quest III. I couldn’t be happier that it is getting an official HD2D remake. There are a few differences with how I use the art style though. I don’t rely on post-processing effects outside of basic colour correction. While bloom can help simulate the light bleed of CRT displays, the excessive depth of field blur for the tilt-shift effect should be used sparingly, and doesn’t make much sense unless the game is set in a miniature environment like Pikmin.

![Xenogears-(by-The-Dark-Id)/Update%2009](/assets/img/2023-11-05-perfecting-unitys_billboard_shader_for_hd2d/8-dazil17.jpg)
[Xenogears by The Dark Id](https://lparchive.org/Xenogears-(by-The-Dark-Id)/).

## Billboards in Unity

![Billboard0](/assets/img/2023-11-05-perfecting-unitys_billboard_shader_for_hd2d/L5H7Ypy.png)

Achieving the HD2D art style in Unity was not as simple as I expected. There are several effects that I wanted to have on the characters, including surface normals, shadows, perspective projection and a dissolve effect. Using Unity’s sprites causes issues in a 3D environment due to sorting, especially when it comes to transparent effects like water, and does not support normal maps.

![Editor1](/assets/img/2023-11-05-perfecting-unitys_billboard_shader_for_hd2d/Editor1.png)

The Billboard Renderer is intended for trees and scenery at their highest LOD (Level of Detail) so the easiest way to start using it to create or import a tree asset and remove all of the 3D models, leaving just the billboard.

![Editor0](/assets/img/2023-11-05-perfecting-unitys_billboard_shader_for_hd2d/Editor0.png)

This also gives you the LOD Group component which can be used to fade out the billboard when it is too close or too far from the camera. However, if you want more control like being able to fade out the character upon defeat, this can be done in the shader instead.

![Editor3](/assets/img/2023-11-05-perfecting-unitys_billboard_shader_for_hd2d/Editor3.png)

The Billboard Renderer requires a Billboard Asset which, when properly configured, takes a texture with the character facing multiple directions and displays the correct one based on the angle they are facing and the angle of the camera. Other billboard implementations have to do this in a script or a shader, which can make things more complicated if you want to support billboards with different numbers of angles.

![Editor2](/assets/img/2023-11-05-perfecting-unitys_billboard_shader_for_hd2d/Editor2.png)

With a Billboard Asset you just need to tell it how many angles are in the sprite sheet and the UVs and it will do the work for you. Animating characters is as simple as adding an offset to the UVs each frame. You can also set the width and height, as well as a vetical offset to ensure that the character is aligned properly with the ground.

![Billboard1](/assets/img/2023-11-05-perfecting-unitys_billboard_shader_for_hd2d/GIJd6qL.png)

The main issue with the Billboard Renderer is what happens when you look at the billboard from above. The billboard becomes razor-thin because only the X and Z angles are rotated to face the camera. The same issue applies to shadow rendering, where the shadow becomes a thin line when the sun is directly above, since shadow maps are typically created from a camera that captures the depth instead of the colour. This can be improved by adding a decal shadow but it will not hide the real shadow being cast. Billboards will also receive their own shadows, so unless you’re very good at authoring normal maps or you're using a more recent version of URP that supports shadow layers, they’re almost always going to be half in shadow.

![Editor4](/assets/img/2023-11-05-perfecting-unitys_billboard_shader_for_hd2d/Editor4.png)

If you’d also like to use billboards for terrain details like trees, you can enable BILLBOARD_FACE_CAMERA_POS in the quality settings. This rotates billboards in such a way that two adjacent billboards will not overlap each other as you rotate the camera. However, their shadows will also be affected by this rotation meaning that the width of an object’s shadow can vary drastically just by rotating the camera.

## Solving the perspective issues

At this point, if you’re not familiar with writing shaders or the 3D trigonometry involved, then you’ll either need to accept these flaws or start looking for another solution. But what if we could change the shader to behave how we want it to?

Anyone working with URP should become familiar with [ColinLeung-NiloCat](https://github.com/ColinLeung-NiloCat) whose repos are fantastic examples for many visual effects that are also optimised for mobile devices. [UnityURP-BillboardLensFlareShader](https://github.com/ColinLeung-NiloCat/UnityURP-BillboardLensFlareShader) is a billboard shader that is always facing the camera, so by reproducing what the vertex shader does in Unity’s billboard shader we should get the same effect.

First of all, you should create copies of the following files from /Packages/com.unity.render-pipelines.universal/Shaders/Nature/ in your project and rename them. If you aren’t using the latest version of Unity then you may want to get them from their [Graphics repo](https://github.com/Unity-Technologies/Graphics/tree/master/Packages/com.unity.render-pipelines.universal/Shaders/Nature). I’d suggest removing all instances of the word speedtree, including inside the files.

* SpeedTree7Billboard.shader
* SpeedTree7BillboardInput.hlsl
* SpeedTree7BillboardPasses.hlsl
* SpeedTree7CommonInput.hlsl
* SpeedTree7CommonPasses.hlsl

In your copy of BillboardPasses.hlsl, you’ll want to comment out lines 39 and 40:
```hlsl
input.vertex.xyz += billboardPos;
input.vertex.w = 1.0f;
```
billboardPos is calculated to rotate the billboard to face the camera on the x and z axes, but we’re going to move this to the vertex shader. You can also comment out the lines setting up billboardPos and widthScale and weightScale.
Then you should comment out line 105:
```hlsl
output.clipPos = vertexInput.positionCS;
```
and replace it with this from [URP_NiloCatExtension_BillboardLensFlare.shader](https://github.com/ColinLeung-NiloCat/UnityURP-BillboardLensFlareShader/blob/master/URP_NiloCatExtension_BillboardLensFlare.shader):
```hlsl
// Make quad look at camera in view space
float3 quadPivotPosVS = TransformWorldToView(vertexInput.positionWS);
// Get transform.lossyScale
float2 scaleXY_WS = float2(
length(float3(GetObjectToWorldMatrix()[0].x, GetObjectToWorldMatrix()[1].x, GetObjectToWorldMatrix()[2].x)), // scale x axis
length(float3(GetObjectToWorldMatrix()[0].y, GetObjectToWorldMatrix()[1].y, GetObjectToWorldMatrix()[2].y))); // scale y axis
scaleXY_WS *= unity_BillboardSize.xy;
float3 posVS = quadPivotPosVS + float3(input.texcoord.xy * scaleXY_WS, 0); // Reconstruct quad 4 points in view space
posVS.x -= scaleXY_WS.x * 0.5; // Centre the billboard
posVS.y += unity_BillboardSize.z;
// Complete SV_POSITION's view space to HClip space transformation
output.clipPos = mul(GetViewToHClipMatrix(), float4(posVS, 1));
```
You should repeat this for anywhere else output.clipPos appears.

The main difference from the code in URP_NiloCatExtension_BillboardLensFlare.shader is how it calculates the billboard's position in View Space.
```hlsl
float3 quadPivotPosOS = float3(0,0,0);
float3 quadPivotPosWS = TransformObjectToWorld(quadPivotPosOS);
float3 quadPivotPosVS = TransformWorldToView(quadPivotPosWS);
```
has been changed to
```hlsl
float3 quadPivotPosVS = TransformWorldToView(vertexInput.positionWS);
```
This is because if the position in Object Space is set to the origin (x=0, y=0, z=0) then even if you move the billboard GameObject the character will not move from that position. Instead, we need to get the vertices in World Space vertexInput.positionWS. This is why we needed to avoid changing input.vertex.xyzw before.

The other difference is the translation and scaling done to the billboard. This uses the width, height and offset from the Billboard Asset so your character should be rendered with the correct scale and position. However, the scale factor applied to the GameObject will now have no effect.

In the next part, I'll go through how to improve the shadows.

[back](/)
