---
layout: post
title: "A hybrid rendering pipeline for realtime rendering: (When) is raytracing worth it?"
date: 2019-03-26T17:50:23+00:00
original_url: https://auzaiffe.wordpress.com/2019/03/26/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/
---

First of all, let’s be clear about this: in this blog I am only expressing **my personal opinion (not my employer’s)**. As a result of my experience, it is then by definition subjective (sometimes even wrong). However, the goal here is to rationalize it.

Okay, let’s talk about the herd of elephants in the room.

![Two elephants standing in a hotel lobby](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-c9f9e9-0_0_3400293_01_640x333.jpg)

## Trendy
Today (25 March 2019) is a very trendy time to be doing raytracing in games/interactive experiences.

Nvidia is pushing **hard** its usage in the game industry: Ads everywhere, plenty of GDC talks, demonstrations at their conference (GTC), a book that is there to convince you to add it to your game production and even better you could win that same book at almost all their GDC talks.

![Jensen Huang on stage presenting the GeForce RTX 2080](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-c5469e-nvidia-geforce-rtx-reveal.jpg)

*Jensen Huang Nvidia’s CEO presenting the RTX line GPUs*

They have massively invested in making this feature a trend, and one would even say that it is a successful campaign. At GDC, there were plenty of talks about developers explaining how they did their integration, or how developers should be doing it. Game developers (including Dice, 4A, Eidos, and many others) went down the “RTX ON/OFF” road at last minute on big IPs to showcase how great it is. Major game engines (Unity/Unreal) also invested in it. Even AMD announced that they were working on raytracing support for their GPUs.

![NVIDIA slide showing the games with upcoming RTX support](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-32ecd0-Nvidia-GeForce-RTX-games.jpg)

*Game titles that have been announced to use raytracing*

The press coverage is also impressive. Raytracing is advertised as “simple” and solving all the problems we have been tackling forever in 3D graphics development. Everyone can understand it, and it almost sounds like if you buy an rtx card or use the raytracing APIs, 3D rendering solves itself. **Spoiler alert: it doesn’t.**

Don’t get me wrong, I think it is amazing that we are in an era where we can have GPU hardware that is able to run BVH traversal in **such short time on such high complexity** scenes (geometry, materials and lighting). It is for sure a first step towards having it part of the rendering pipeline apis/tools and there are useful things to be done with it. However, I think the over-excitement that we are seeing as game developers is frustrating for reasons that I’ll try to express in this post series.

## Reach
Raytracing dedicated hardware is a PC only, high-end devices, **Nvidia feature**. Meaning that even if it is available in the Vulkan and DX12 standard APIs, only an infinitely small percentage of users are going to take advantage of it. That simple fact makes investing in it a complex decision for the majority of game developers (especially small and average sized ones).

![The AMD and Intel logos](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-1561c2-intel-vs-amd.png)

*Two major vendors haven’t released raytracing hardware (yet)*

Yes, I am aware of the recent announcement to add support of it on previous Nvidia GPUs ([link](https://www.theverge.com/2019/3/18/18271836/nvidia-raytracing-gtx-rtx-older-gpu-support)). In practice, they are announcing that they will be supporting the fallback layer that they dropped in [November 2018](https://github.com/Microsoft/DirectX-Graphics-Samples/tree/master/Libraries/D3D12RaytracingFallback) (given that there is no dedicated BVH traversal hardware on those).

![The D3D12 Raytracing Fallback Layer documentation](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/fallbacklayer.png)

In game development best practices, I think extensions are to be avoided whenever possible (I am not including consoles and I am not limiting this statement to GPUs). If they imply big changes, a lot of work on the developer side, and on top of that, most of users are not going to take advantage of them, there are only a few reasons that would push devs to use them.

Do not forget that you can’t test them as much as other features because they are usually tied to a specific set of hardware and most test beds are not designed like that. That adds complexity to the process of integrating them into a game production.

Here are the few reasons that I think would push developers of doing it despite all the previous things:

- The extensions are console extensions and then there is a huge amount of users that are going to be impacted
- The developer has a personal interest into making the feature part of the standard gamedev tools
- The developer is trying to make a technological statement
- The developer has a partnership with the vendor
- The developer has a partnership with API developer

I am not even mentioning that we haven’t heard about any plan of supporting this on platforms that have the biggest reach for gamedevs: **consoles**. Game developers will start to figure out the best usage of this technology when the millisecond clock is going to knock.

![An Xbox One S and a PlayStation 4 Slim](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-d6ec05-ps4-slim-vs-xbox-one-s-555x328-1143571.jpeg)

## Integration
Let’s ignore what was mentioned before and suppose that everyone is convinced of using it, I am sorry to say: **it doesn’t “just work”**. Pretty much everyone that worked on it so far has reported the difficulties of making it part of an actual game dev production.

![Jensen Huang beside the quote "It just works"](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-582585-maxresdefault.jpg)

A general problem with the D3D12 API is that it is pain to integrate into a pre-existing production ready engine. The lower-level paradigms that D3D12 brings to the table are often incompatible with the ones that D3D11 implementations have. Some implementations might even end up with higher frame costs while using a lower level API.

Don’t get me wrong, **I think the new reach of graphics developers is really good to have** and in theory gives access to higher performance, but that doesn’t change the struggle that it is in practice for game developers to write cross APIs abstractions.

On the other side, the current raytracing APIs is a high level one: very little is exposed to the developers (BVH structures are opaque, no control over the recursive *TraceRay* coherence and dispatch, cheating is required to support skinned meshes and particles, transparents are a nightmare, material LOD along recursion is hard, etc.).

![Diagram comparing the rasterization and ray tracing pipelines](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-2efc32-nvidia_rtx_vulkan_med.webp)

*Simplfied Raster and Raytraced Pipelines compared by Nvidia*

While it is probably the best thing to do for an initial launch, it is still a new API and everyone is expecting it to take some time before reaching the maturity of the already existing graphics APIs.

That said, the paradigms that raytracing implies on an engine are pretty different from the ones that rasterization does. While it is possible to do an integration, it changes a lot in terms of what resources should be available at what time (rendering states, per draw data, material LODs, geometry LODs etc.). That means that in order to make it work, you need to re-think parts of your engine.

The spatial partitioning structures that are already in place become obsolete and alternatives are to be found (frustum based partitioning, shadow cascades, geometry culling, light culling, etc). All the data that are camera/raster dependent also become problematic to handle (camera relative rendering, mip selection, DDX/DDY, procedural vertex animation, etc).

That said, there are solutions (more like workarounds) for all the things that have been mentioned in this section. I am just trying to point some of the struggles that developers are facing when trying to make this part of their engine.

## Cost
Obviously, there is a reason why game developers didn’t use raytracing before in this context and it is not just because there was no API for it. **Multi-sampled, full screen raytracing and shading is bloody expensive**.

![A meeting room rendered with ray-traced primary visibility](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/raytracedprimary9-1.png)

*Nine bounces to achieve convincing rendering for the refractive bottles in Unity HDRP*

While in theory the complexity of rasterization is O(N) and Raytracing is O(Log(N)) (if we ignore recursion), the inherent cost of bvh traversal and cache incoherency plays a lot in the process in making harder the latter one to scale (unless everything is perfectly smooth and thus there is no ray diffusion, which is not the generic use case).

In addition to that, hardware and software engineers have converged over the years to GPU architectures and usage that have an efficiency that is very hard to compete with. We’ve become really good in rendering infinitely complex data while having tools to measure everything to understand how to make it optimal.

GPU raytracing API / architecture is new to the realtime game, while I believe that **it brings something** that is hard to achieve otherwise, it is inefficient in its primitive form and it is a challenge for it to be able to compete with rasterization in some parts of the rendering pipeline.

Another thing worth mentioning is the denoising/filtering solutions that Nvidia is offering as out of the box. While the result is looking good, unfortunately a single profiling of it forces the developers to look for cheaper/more scalable solutions. A screen space pass that takes 3ms on a very high end graphics card (2080 Ti) is too much, and I am not even talking about the machine learning filters that are not 100% reliable (having in mind that the full rendering can take less than 3ms on those cards).

![Quake II with ray tracing: raw path-traced output beside the denoised result](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-45c585-small_Quake-2-Ray-Traced-path-tracer-denoised.jpg)

*Pre/post denoising of an image in the “path traced” Quake 2 using SVGF*

Raytracing also implies a lot of additional render target/resource allocations which is already a struggle in current render pipelines without raytracing. Developers then need to be even more careful with their resource management and re-use in order to make it acceptable.

Raytracing profiling/validation tools are just starting to be a thing, Nvidia announced at GDC the full set of features that they will be adding to Nsight Graphics in order to measure the various costs that a given *DispatchRays* implies and that will help making the usage of the raytracing APIs a real thing in game production. **I am happy to have that from now on!**


*Screenshot of Nsight graphics for the raytracing acceleration structure (RAS) view*

## Usage
The global perception that I feel for the moment is the following one: Raytracing is here and it is going to solve all the hacky things that we had to do to make rendering possible. **That is simply not true.**

While it simplifies a lot of things and makes them more physically accurate, reaching the quality of productions that are not using rasterization would mean launching a number of rays that is just incompatible with the notion of realtime/interactive.

Note that even in offline rendering, some effects are not done using raytracing because it increases significantly the frame cost or constraints too much the compositing process (Depth of field).

Right now, most of the developers that integrated raytracing based effects, implemented effects that are the cheapest possible while tackling problems that are very complex to resolve using other approaches (I didn’t say impossible). A non-exhaustive list would be:

- Rough reflections (Accurate indirect specular)
- Ambient occlusion
- Indirect diffuse (or/and rough indirect specular)
- Area light visibility (or shadows depending on the implementation)
- Directional light shadows (as an infinite disk light)

![Ray-traced reflections in a dimly lit meeting room](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/raytracereflections.png)

Raytraced reflections in Unity HDRP (1SPP +  Filtering)

Other effects have been demonstrated, but I would say right now these are the ones that have been investigated by game developers while keeping performance in mind.

What I am trying to say here is: For developers that spent extensive time during a raytracing variant of these effects, the viable solution lies somewhere in between the screen-space/rasterization based approach and the full raytraced pass (I recommend the SEED and Frostbite talks on raytraced reflections and the Metro Exodus Talk on Indirect Diffuse/Indirect Specular, links at the bottom of the page).

Developers are just starting to figure out what would be the best usage of this API with all the knowledge that they have accumulated through the years.

I’d say right now (and I hate to say this), the raytracing trend feels a lot like the machine learning trend. An old technology became popular again, and everyone states that it will solve all the problems of the world (and it will not). However, there are **some configuration where there is a clear win**.

On the other side there are many configurations where rasterization techniques are more practical and more efficient.

![Planar reflections on the floor of an interior scene](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-504052-planarReflections_Banner.png)

*Planar reflections in [UE4](https://docs.unrealengine.com/en-us/Engine/Rendering/LightingAndShadows/PlanarReflections)*

![A side-by-side comparison of shadow softness under a car](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-d3e198-0T3a3cp.png)

*Soft sun shadows using PCSS – Need for Speed*

Something that I intentionally did not talk about, is the viability of all of this in a **semi-interactive context**. For producing movies/video parts for instance. The real-time constraint can be relaxed to a certain point (it can go up to a few seconds per frame when rendering in 4k/8k). It is then possible to **achieve a lot** while using a hybrid pipeline rendering pipeline.

## Art Direction
Raytracing has a lot of impact on the art direction of the game. More physically accurate, is not always what artists are looking for.

![An "RTX off" and "RTX on" meme using two cartoon renderings of the same character](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-b29f68-5gwo9dcm1rh11.jpg)

Giving the timing of the raytracing integration that game developers had to deal with (late into the production state), it was really hard for them to keep intact the art direction that was defined during pre-production state.

Some of the replacements (light probes versus raytraced indirect diffuse for instance) change drastically the look of the game. We can then end up with a game not looking better with raytracing-based effects but just looking different (or sometimes worse).

![Metro Exodus shown with ray tracing off and on](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/rtxonmetro.png)

*Character looking too dark with raytraced indirect diffuse in Metro Exodus*

Also I saw a lot RTX ON/OFF comparisons where developers are making everything shiny to demonstrate the tech while it does not serve the content.

![Ray-traced reflections on armour in Justice](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/justicereflections.png)

*(too) Smooth raytraced reflections in Justice*

![A cartoon scene labelled "RTX off" and "RTX on"](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-50542b-829.jpg)

*Smooth reflections in spongebog.*

I guess this one will be easier to handle in upcoming game productions. Developers and artists will have the raytracing option in mind during the pre-production of the game and will make sure that it is interesting for them to use. If it does not bring anything to  your game, do not use it.

## Worth it?
I would say raytracing hardware **definitely brings something to the table.** While not being a magical solution, it offers new perspectives that will change, on the long term, how we do things in realtime rendering. There are things to take right now for interactive experiences and lots to build for real-time ones.

I may have sounded hopeless, but that is far from being my current state of mind. I think this is just the beginning and the exciting part is ahead of us. Papers that take advantage of this new API in a smart way are starting to pop, and **I am pretty****excited with the all the possibilities that this offers**.

![The cover of the book Ray Tracing Gems](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/ext-e6f23b-ray-tracing-gems.jpg)

*Recently released Ray Tracing Gems, link at the bottom of the page*

We have been experimenting in HDRP with this new API, If you are interested, the next thing that I will be covering is the raytracing “**Integration into HDRP**“.

![A ray-traced car interior](/assets/img/posts/a-hybrid-rendering-pipeline-for-realtime-rendering-when-is-raytracing-worth-it/image-2-1.png)

*Screenshot from the BMW Unity Realtime raytracing Demo*

If you made it to here: thanks for reading and I hope you learned something or at least enjoyed it.

Thanks to Francesco, Julien and Sebastien for proofreading this post!

**References:**

[DD2018: Tomasz Stachowiak – Stochastic all the things: raytracing in hybrid real-time rendering](https://www.youtube.com/watch?v=MyTOGHqyquU)

[IT JUST WORKS: RAY-TRACED REFLECTIONS IN ‘BATTLEFIELD V’ –](https://schedule.gdconf.com/session/it-just-works-ray-traced-reflections-in-battlefield-v/863651)[Jan Schmid](https://schedule.gdconf.com/speaker/schmid-jan/54840), [Johannes Deligiannis](https://schedule.gdconf.com/speaker/deligiannis-johannes/46966)

[EXPLORING THE RAY TRACED FUTURE IN ‘METRO EXODUS’ (PRESENTED BY NVIDIA)](https://schedule.gdconf.com/session/exploring-the-ray-traced-future-in-metro-exodus-presented-by-nvidia/865258)[Oles Shyshkovtsov](https://schedule.gdconf.com/speaker/shyshkovtsov-oles/54025), [Ben Archard](https://schedule.gdconf.com/speaker/archard-ben/54026), [Dmitry Zhdan](https://schedule.gdconf.com/speaker/zhdan-dmitry/39798), [Sergei Karmalsky](https://schedule.gdconf.com/speaker/karmalsky-sergei/54664)

[ADVANCED GRAPHICS TECHNIQUES TUTORIAL: “SURFING THE WAVE(FRONT)S WITH RADEON GPU PROFILER” & “DEBUGGING AND PROFILING DXR & VULKAN RAY TRACING”](https://schedule.gdconf.com/session/advanced-graphics-techniques-tutorial-surfing-the-wavefronts-with-radeon-gpu-profiler-debugging-and-profiling-dxr-vulkan-ray-tracing/865030)[Dominik Baumeister,](https://schedule.gdconf.com/speaker/baumeister-dominik/53851) [Aurelio Reis](https://schedule.gdconf.com/speaker/reis-aurelio/30406)

[Spatiotemporal Variance-Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination](https://research.nvidia.com/publication/2017-07_Spatiotemporal-Variance-Guided-Filtering%3A)

[Ray Tracing Gems – Real-Time Rendering](http://www.realtimerendering.com/raytracinggems/)

[Real-Time Ray Tracing of Correct* Soft Shadows  – Eric Heitz](http://casual-effects.com/research/Heitz2018Shadow/Heitz2018SIGGRAPHTalk.pdf)

[RAY-TRACED WATER CAUSTICS WITH DXR (PRESENTED BY NVIDIA) –](https://schedule.gdconf.com/session/ray-traced-water-caustics-with-dxr-presented-by-nvidia/865252)[Holger Gruen](https://schedule.gdconf.com/speaker/gruen-holger/28064)
