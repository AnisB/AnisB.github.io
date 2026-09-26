---
layout: post
title: "Currents in HDRP's Water System"
date: 2026-09-26T05:31:39+00:00
tags: [computer-graphics, games, real-time, rendering, technology]
math: true
---

## Introduction

I always want to come back to water rendering. Of all the rendering systems I have worked on it is the one that gives back the most visual feedback (that and clouds). You change a value and the surface immediately looks different, and you can tell whether it is better just by looking at it. That loop is probably why people get into rendering in the first place.

The remaining members of the original HDRP team left Unity this summer, and that event sent me back to the code. I opened it with no particular goal, ended up back in the water current system, and stayed there longer than I meant to.

The approach is one I came up with when I designed HDRP's water currents, and as far as I know it is not how currents are usually done. Apologies if I have missed prior work. It was never written up, and somehow the last time I touched it was four years ago, so this is an attempt at documenting it.

Currents let the water flow in a direction that varies over space, so a river bends around an island, a vortex spins behind a pier, and the open sea a hundred meters away carries on behaving like open sea.

By the end of this post we will have built this, and nothing in the water system will know what a vortex is:

<figure class="clip">
<video src="/assets/img/posts/currents-in-hdrps-water-system/vortex.mp4" autoplay loop muted playsinline></video>
<figcaption>A vortex on an open ocean surface. Two channels of one decal, and the simulation doing the rest.</figcaption>
</figure>

What made it fun to build was never the maths that makes it work. It was the range of things that turned out to be buildable with it.

Everything here lives in `Runtime/Water` of the [HDRP repository](https://github.com/Unity-Technologies/Graphics), if you want the details. For the wider picture there is the [blog post from when the water system shipped](https://unity.com/blog/engine-platform/new-hdrp-water-system-in-2022-lts-and-2023-1), which covers the system as a whole rather than this one corner of it.

## Why this is harder than it sounds

The simulation underneath is the well beaten path: a Tessendorf style oceanographic spectrum, an inverse FFT, and the result written into a tiling patch. That model is a statistical description of open water driven by wind. It has no notion of a bank, an obstacle, or anywhere the water is supposed to go.

![The displacement and normal buffers the FFT produces for each band](/assets/img/posts/currents-in-hdrps-water-system/simulation-buffers.png)
*What the simulation produces. Displacement is the FFT output, one tiling patch per band, with that band's normals below it.*

The usual implementation gives each band a single global orientation. That is enough to express surface wind running across a longer swell, which is worth having and is most of what an ocean needs, but it is still one direction for the entire body of water. There is nowhere in it to put a flow that changes from place to place.

What we wanted was the opposite of a special case. Not a river feature and a waterfall feature and a vortex feature, each with its own parameters and its own code path, but one data oriented mechanism: a field the surface reads and knows nothing else about. Paint it by hand, generate it in a Shader Graph, bake it offline, simulate it at runtime, it should not matter. The water samples a direction, and that is the entire contract between it and whatever produced the field.

That requirement rules out more than it sounds like it does. Currents are asking the simulation for something it was never built to express, and there are two obvious ways to ask, both of which fail.

**Rotate the sampling and read it back.** Take a direction from the current map and rotate the sampling coordinates by that angle. For one constant angle this is fine: a rigidly rotated patch is still a perfectly good wave field. The trouble is that the rotation is applied to position, and the angle varies from place to place. The sampled coordinate is then a rotation that changes with position applied to that position, and a small change in the angle moves it by an amount proportional to how far the point is from the origin. Far enough out, neighbouring points sample unrelated parts of the patch, and the surface swims and tears everywhere the direction is changing, which with a continuous field is everywhere.

**Do it properly instead.** Take the field into the spatial domain, apply constraints where the water meets geometry, let the boundaries push the flow around the way a solver would, and transform back. This is the honest answer and it produces genuinely correct behaviour. Evan Wallace's [WebGL Water](https://madebyevan.com/webgl-water/) has been the reference for how convincing it can look for years, and it is worth noticing that it is a pool. It also does not scale. The cost is tied to the area you have to cover at a resolution fine enough to matter, and doing that round trip every frame is affordable for a pool and nothing larger.

Medium and large bodies of water are exactly what the system has to support. An ocean, a river running through a level, a lake you can see the far side of. The approach that works beautifully in a bathtub is the one you cannot ship, and that constraint is what pushed the solution towards something that costs a sample rather than a simulation.

There is a second saving hidden in that. Because currents never touch the simulation, the simulation stays generic, which means water bodies of the same nature can share one. A set of rivers running the same spectrum is a set of current maps and meshes reading from one simulation, not one simulation each. The approach that constrains the field against geometry cannot do that: the constraints belong to a particular body, so the simulation does too, and the cost grows with how many bodies of water are in the level instead of staying flat.

The rest of this post is about the approach that came out of those constraints. What it is like to use first, then how it works underneath.

## How to feed the water current system: a texture

The simplest way to make a current is to paint one. A current map is an ordinary texture, imported with sRGB off:

- **Red and Green** hold the 2D direction of the flow.
- **Blue** holds the influence, meaning how strongly this pixel overrides the surface's base direction.

![How a current map encodes direction and influence](/assets/img/posts/currents-in-hdrps-water-system/current-map-encoding.svg)
*Two channels for direction, one for influence, and a neutral value you can paint on top of.*

The neutral value is `(1, 0.5, 1)`, and the default direction is +X. That last detail matters more than it sounds: because neutral is a real colour rather than black, you can start from a flat fill and paint deviations into it, and anywhere you have not painted simply keeps flowing the way the rest of the surface flows.

The influence channel is what makes this pleasant to author. It is not a mask that turns the current on and off, it is a blend against the underlying motion. You can paint a river that grips the water hard in the middle of the channel and lets go towards the banks, and the transition costs you nothing because it is one channel of the texture you are already painting.

Resolution barely matters. The direction field is low frequency by nature, and the simulation detail comes from the FFT bands rather than from the map, so a 512 is usually more than enough.

To see what you have actually painted, the surface has a debug mode that draws the flow field as arrows directly on the water. There is a small arrow SDF in the shader for exactly this:

```cpp
#define ARROW_TILE_SIZE 4.0

float EvaluateArrow(float2 positionAWS, float2 dir, float2 tileSize)
{
    float2 tileCoord = frac(positionAWS / tileSize) * 2.0 - 1.0;
    float x = dot(dir, tileCoord);
    float y = dot(float2(dir.y, -dir.x), tileCoord);
    float mask = 0.0;
    // Arrow body
    if (y > -0.1 && y < 0.1 && x > -0.9 && x < 0.5)
        mask = 1.0;
    // Arrow head
    else if (y > -0.4 && y < 0.4 && x > 0.5 && x < 0.9 && (0.4 - (x - 0.5)) / abs(y) > 1)
        mask = 1.0;
    return mask;
}
```

Being able to see the field you are authoring, in the scene, at the scale you are authoring it, is most of what makes a feature like this usable.

## Swell and ripples are independent

A water surface does not have one current, it has two. The large waves and the ripples carry their own direction field, their own region, their own influence.

This is not a generalisation for its own sake. Wind-driven ripples genuinely do run across a river while the body of the water moves downstream, and if you force them to share a direction you lose the thing that makes moving water read as moving water. Where they should agree, the ripples can be set to inherit from the swell (or from the agitation, on a river surface) and you author one map instead of two.

Which band a given current drives depends on the surface type: on an ocean you set the map under **Swell**, on a river under **Agitation**, and on a pool you only get **Ripples**, because a pool with a swell is not a pool.

## One decal, every channel

Currents are a channel of the water decal system. The same Shader Graph Master Stack that writes deformation and foam also writes current:

![The channels a single water decal can drive](/assets/img/posts/currents-in-hdrps-water-system/decal-channels.svg)
*A single water decal, projected in world space, writes into deformation, foam, masking and current in the same frame over the same region.*

The blocks are enabled per graph, so a decal only pays for what it uses:

| Block | What it does |
| --- | --- |
| `Deformation` | Elevation added on top of the surface in the decal area |
| `HorizontalDeformation` | Horizontal offset in UV space |
| `SurfaceFoam`, `DeepFoam` | Foam on the surface and in the volume |
| `SimulationMask` | Where the simulation applies at all |
| `LargeCurrent` + `LargeCurrentInfluence` | Swell flow direction, and how strongly it overrides the base flow |
| `RipplesCurrent` + `RipplesCurrentInfluence` | The same for ripples |

Consider what that composition buys you. A boat wake is not a current effect, or a deformation effect, or a foam effect. It is one decal that pushes the surface down, throws foam along the crests, and drags the flow outwards behind the hull, authored in a single graph, moving with the boat. A waterfall base is one decal that deforms, foams, and pushes the current radially away from the impact point. You are not synchronising three systems and hoping they agree, because they are one system with several outputs.

Because the decals are projected in world space and can be anchored to a GameObject rather than the camera, they move with whatever produced them. And because they are Shader Graphs, the flow field can be computed rather than painted: a spline, a distance field, anything you can express in a graph.

## Currents that accelerate

Current maps have a real limitation, whether painted or written by a decal: the direction varies over space, but the speed does not. The flow cannot stop and it cannot speed up. For an open ocean that is invisible. For a river narrowing into a gorge it is exactly the thing you need.

It is worth being precise about this, because the naming suggests otherwise. The `LargeCurrent` block takes a float2 and the documentation describes it as direction and strength, but what actually reaches the simulation is an angle:

```cpp
float3 dir = float3((cmpDir.xy * 2.0 - 1.0) * flipDir, cmpDir.z);
float angle = ConvertAngle_NPI_PPI(atan2(dir.y, dir.x) - orientation);
if (dir.x == 0 && dir.y == 0) angle = 0;
return angle * influence * dir.z;
```

The vector goes through `atan2` and its length is thrown away. No amount of magnitude in the map will make the water move faster. The influence channel scales the angular deviation away from the surface's base direction, which is a blend, not a speed.

Speed comes from somewhere else entirely, and it is not in the currents at all, it is in the mesh. An infinite water surface derives its sampling coordinates from a planar XZ projection, world position straight into the simulation, which makes uniform scrolling a property of the projection: every point advances at the same rate because the mapping says so. But a surface does not have to be infinite. Give it a custom mesh and you author the UVs yourself, and the simulation is read along those instead. Pack the UVs tighter through a section and the water crosses it faster, stretch them and it slows down.

So acceleration is not something the simulation models, it is something the geometry decides. You lay out a mesh whose UV density matches how fast the water should be moving at each point, and the flow speeds up and eases off because the coordinates it is being sampled at do.

![A mesh unwrap with one stretch squeezed in UV, and the same quads on the ground](/assets/img/posts/currents-in-hdrps-water-system/uv-unwrap.svg)
*Left, the unwrap, with one stretch of the channel given less UV than its neighbours. Right, the same quads on the ground, evenly spaced. The pattern is uniform in UV and stretched in world space exactly where the unwrap was squeezed, so it sweeps past faster there.*

<figure class="clip">
<video src="/assets/img/posts/currents-in-hdrps-water-system/cascade.mp4" autoplay loop muted playsinline></video>
<figcaption>A cascade dropping into a pool. The water accelerates through the fall and settles below it, and a decal adds the foam that should come with it.</figcaption>
</figure>

## Three things you can build with it

The three cases below each lean on a different part of the system.

![Three flow fields: a river bend, an accelerating cascade, and a vortex](/assets/img/posts/currents-in-hdrps-water-system/flow-fields.svg)
*The fields behind each one, and what each needs. The river varies direction only. The cascade varies speed, which comes from the unwrap rather than the field. The vortex turns the flow and pulls the surface down at the same time.*

**A river that follows.** The simplest case. Direction varies, speed does not. A painted map bends the flow along the channel, and the influence channel lets it grip hard mid channel and relax towards the banks.

![The river current map, the debug view of the flow field, and the result](/assets/img/posts/currents-in-hdrps-water-system/river-breakdown.png)
*The map on top, painted to follow the channel. Below it, the same surface in current debug mode, and the result beside it. In the debug view the arrows and the colour both encode local orientation, so the red stretch is where the river turns to head in a different direction, not where it runs faster.*

Painting the map is only the simplest way in. Because the current is read from a texture, anything that can write a texture can drive it. Simulate the flow from a source texture and an input current, advect particles through it and render their velocities out, and the river follows whatever came out the other end. The surface does not care how the field was produced, only that there is one to sample.

<figure class="clip">
<video src="/assets/img/posts/currents-in-hdrps-water-system/river.mp4" autoplay loop muted playsinline></video>
<figcaption>The same river running. The simulation is unchanged, only the direction it is sampled along.</figcaption>
</figure>

**A cascade that accelerates.** This one needs speed to vary, which no current map can do, so the mesh does it: the unwrap is compressed through the drop. The water quickens over the fall and eases off downstream, and because a decal writes the deformation and the foam over the same stretch, the acceleration arrives with the whitewater that should accompany it. The clip in the previous section is this case.

**A vortex.** The one that best shows why the design is shaped this way. It is two channels of one decal: a current field that rotates about a point, and a vertical deformer pulling the surface down at the centre. Neither half is convincing on its own. Rotation with a flat surface reads as a texture spinning, and a funnel with no rotation reads as a hole. Together they read as a vortex, and the simulation supplies every bit of the detail.

That is the argument for putting currents in the decal stack, in one example. Nothing in the system knows what a vortex is. There is no vortex feature, no vortex parameters, no vortex code path. It falls out of two general channels being authorable in the same place, which is usually a sign the mechanism is general enough.

And because it is just a direction field, you can generate the map rather than paint it. Here is the whole thing end to end: the current map that rotates the flow, the deformation that pulls the surface down, and what the two of them produce together.

![A vortex current map, its deformation map and cross-section, the resulting render, and its deformation buffer in HDRP](/assets/img/posts/currents-in-hdrps-water-system/vortex-breakdown.png)
*The two inputs on top, the result below. In the current map the hue is the direction. In the deformation, mid grey is flat and the cross-section shows the profile: a funnel, and the small lip that rises just before the surface falls away. Bottom right is the same decal as HDRP sees it in the deformation buffer: an elevation added on top of the simulated surface, separate from the FFT.*

That lip is worth the trouble. Without it the funnel meets the surrounding water flat and reads as a hole cut into the ocean. With it, the water bulges slightly before it drops, which is what a real vortex does and what makes the eye accept the rest.

## How it works underneath

Back to the constraint from the start. The rotation tears the surface because it varies continuously with position, so the fix is to stop it varying: snap the direction to a fixed set of angles, so that within each sector the rotation is rigid. At exactly those angles the result is not an approximation at all, it is a single tap of the field rigidly rotated, with precisely the statistics the simulation was built to have. So the direction is quantised into eight sectors, and the fractional position inside the sector is kept:

```cpp
#define NUM_SECTORS 8u
#define SECTOR_SIZE ((2.0 * PI) / NUM_SECTORS)

float data = angle / SECTOR_SIZE;
float relativeAngle = frac(data);
uint uintData = (uint)data;
currentData.quadrant = uintData > NUM_SECTORS ? uintData - NUM_SECTORS : uintData;
currentData.proportion = PositivePow((currentData.quadrant % 2u == 0)
                                     ? relativeAngle : 1.0 - relativeAngle, 0.75);
```

The simulation is then sampled twice, once along each of the two basis directions bounding the sector, and the results blended by that fraction:

![Quantising to eight sectors and blending two taps](/assets/img/posts/currents-in-hdrps-water-system/sector-blend.svg)
*Eight sectors, two taps, one blend. Each tap is a rigid rotation of the same field, and the blend between them keeps the direction continuous across sector boundaries.*

So the cost is one extra simulation tap per evaluation, in the vertex shader for displacement and again in the pixel shader for gradients.

That is the trade, and it is worth being explicit about what is exact and what is not. On the eight directions themselves the result is exact. Between them, the blend is continuous across every boundary, which is what keeps the surface from popping, but it is an approximation: two weighted samples of the same field do not add back up to one sample of a rotated one, and the 0.75 exponent on the blend weight is a tuned compromise rather than a derived constant. At the middle of a sector it weights the two taps 0.41 and 0.59 rather than evenly. It is not perfect, and it works.

Eight sectors is a choice, not a law. Fewer and the crossfade starts to show. More costs nothing extra in taps but buys nothing you can see, so eight is where it stopped.

![Coverage of arbitrary directions, and the cost of getting it](/assets/img/posts/currents-in-hdrps-water-system/arbitrary-directions.svg)
*Eight directions are exact. Everything between them is a blend of the two neighbours, close enough that an arbitrary flow direction reads as arbitrary rather than quantised.*

What matters visually is that the reconstruction is good enough that you cannot tell where the sector boundaries are. The simulation being sampled is the same in every direction, so blending two rotations of it does not soften or smear the result the way blending two different simulations would. You get the detail of a single tap with the orientation of neither.

None of this is really about water. Any time you want to read a precomputed field along a direction that varies from place to place, you have the same problem, and quantise-and-blend is the same escape from it.

## The part nobody sees

Buoyancy, and anything else that asks for the water height from a script, runs on the CPU. The data it reads comes either from a copy of the simulation replicated on the CPU, or from an asynchronous readback of the GPU buffers that arrives a few frames late, but either way the code that samples it runs in C#. Which means the entire sector scheme exists twice, once in HLSL and once in C# over Burst, and the two have to agree closely enough that a boat does not sink into a wave it is visibly riding.

```cpp
static void SwizzleSamplingCoordinates(float2 coord, int quadrant,
                                       NativeArray<float4> sectorData, out float4 tapCoord)
{
    tapCoord = 0.0f;
    int sectorIndex = quadrant + WaterConsts.k_SectorDataSamplingOffset;
    float4 dir0 = sectorData[2 * sectorIndex];
    float4 dir1 = sectorData[2 * sectorIndex + 1];
    tapCoord.xy = float2(dot(coord.xy, dir0.xy), dot(coord.xy, dir0.zw));
    tapCoord.zw = float2(dot(coord.xy, dir1.xy), dot(coord.xy, dir1.zw));
}
```

Two implementations of the same maths in two languages, kept in sync by hand. It is the least glamorous part of the feature and the one most worth finding a better answer for, and it is what makes a barrel dropped into a river actually travel down the river.

## Bonus

No analysis for this one. I had the project open, I could not resist, and it is just fun to look at.

<figure class="clip">
<video src="/assets/img/posts/currents-in-hdrps-water-system/underwater.mp4" autoplay loop muted playsinline></video>
<figcaption>Underwater, which has nothing to do with currents, but explains why I keep coming back to this.</figcaption>
</figure>

It is a whole stack of features layered on top of each other, and it is only together that they make this part believable.

## Closing

The interesting part is not the sector trick. I find it elegant, but it is nothing revolutionary, and it is only the reason currents were affordable. What made them worth having is that they stopped being their own system and became a channel sitting next to deformation and foam, so a wake or a waterfall is one thing an artist authors instead of three things an engineer synchronises.

The feedback loop from the start of this post is also why the decal decision mattered more than the sector one. The person who decides whether any of this worked is an artist dragging something around a scene and looking at the result, so the thing actually worth optimising was how fast they could try an idea.

If you are building something similar, all of this is public and worth reading. The eight sector trick in particular is not specific to water, and it would be good to see it used somewhere else.

<figure class="clip">
<video src="/assets/img/posts/currents-in-hdrps-water-system/vortex.mp4?end" autoplay loop muted playsinline></video>
<figcaption>A direction field, a depression, and a simulation that knows about neither.</figcaption>
</figure>

## Thanks

To everyone who worked on the water system, both while I was at Unity and after I left. Rémi, Adrien, Maxime and plenty of others. Very little of it stayed exactly the way I first wrote it, and it is better for that.

Several of the scenes and assets used in this post come from the Unity water samples. The art is not mine, only the system underneath it.
