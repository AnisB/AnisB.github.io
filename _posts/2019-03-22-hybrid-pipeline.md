---
layout: post
title: "A hybrid rendering pipeline for realtime rendering: Introduction"
date: 2019-03-22T16:16:47+00:00
original_url: https://auzaiffe.wordpress.com/2019/03/22/hybrid-pipeline/
---

This post is not technical, it is just an introduction to what I’ll be presenting in upcoming ones.

![A car rendered with ray-traced reflections, rear three-quarter view](/assets/img/posts/hybrid-pipeline/carshot2.png)

Last year, I joined Unity technologies as a Graphics Engineer and more precisely the High Definition Render Pipeline (HDRP) team. The ambition was to investigate and see what were the options to benefit from the recently announced hardware that had dedicated pipelines for BVH traversal.

With Sebastien, we did not want to write a hacky demo code that would be thrown away after a show, but we wanted to have a real/maintainable integration with everything else that lives into HDRP and that is already used by developers in production.

[HDRP](https://blogs.unity3d.com/2018/09/24/the-high-definition-render-pipeline-getting-started-guide-for-artists/) is a high end cross platform rendering pipeline that targets high quality production in unity, the exact same code is used for PC, PS4 base and pro, XBOX one and one X, VR devices, offline production and more. Thus there is an extensive set of features that we had to take into account everytime we wanted to make a change for adding a raytracing effect or constraint.

Initially, the work was done in the context of the raytracing video demo (<https://unity.com/ray-tracing>) that was demonstrated initially at GDC and GTC 2019.

![A car rendered with ray-traced reflections against a colourful backdrop](/assets/img/posts/hybrid-pipeline/carshot.png)

In collaboration with Sebastien, I was in charge of implementing the render pipeline integration, the raytracing effects, image quality, the shader/material support and user workflow. I obviously didn’t do the content creation part; shout out to Alexandre, Awen and Aymeric from L&S and Kate and Dany from Unity.

I also won’t be covering the backend part because I didn’t own that.

Later, the real time demo was scheduled and thanks to Mike and Laurent that made it possible.

![A dark interior lit by two screens, rendered with the hybrid pipeline](/assets/img/posts/hybrid-pipeline/image-5.png)

Other people helped with all of this, To name a few: Emmanuel, Francesco, Eric, Arnaud, Joel, Tim, Ionut, Jesper, Tian, Melissa, Natasha.

With a lot of hardwork of these people, we were able to ship and demonstrate something that we were proud of. However, the journey was far from being an easy ride. The three major difficulties were:

- Current high end realtime pipeline are not designed with the raytracing’s constraints in mind (and they should not until it becomes part of the standard game-dev workflow)
- Working with beta and then fresh new drivers is hard. Anyone that ever did it can relate.
- No profiling tools for most of the project, and then very few metrics.

From the various interventions of people that have been doing the same job in other companies, the feedback is pretty much the same.

Pretty much everything that I’ll be describing here and in the upcoming blog posts can be checked in the Scriptable Render Pipeline github repo: <https://github.com/Unity-Technologies/ScriptableRenderPipeline>

As mentioned at the start of this post, this was simply an introduction to the subject. In upcoming posts, I’ll be discussing technical aspects of our implementation. Here are some of the subjects that will be covered:

- When is raytracing worth it?
- Integration into HDRP
- Raytracing effects
- Sampling
- Filtering
- Feature parity between rasterization and raytracing
- Scalability, Cost and Optimization
- Artist workflow
- Profiling and timings
