---
layout: post
title: "Sample Decorrelation in BRDF Preconvolution for Image Based Lighting"
date: 2026-01-10T15:20:10+00:00
original_url: https://auzaiffe.wordpress.com/2026/01/10/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/
tags: [computer-graphics, games, real-time, rendering, technology]
---

## Introduction
During my time as a member of the HDRP team at Unity, one of the systems I contributed to was the split-sum approximation-based image-based lighting, more precisely for the Sheen lighting model (rough cloths), back in 2018.

Recently, while implementing IBL for the engine I’m building at my job, I revisited the HDRP code (as I consider it a very good open-source reference for real-time rendering, which I cannot recommend enough). After having the whole thing working as expected in my new implementation, a weird seam appeared on two of the faces of the convolved LD term (X+ and X-).

At first I thought I had made a mistake, but then I checked a RenderDoc capture of an HDRP frame and realized the same problem was there. Weirdly enough, nobody had complained about it and I don’t recall seeing it in captures at the time, but it seems this issue has been there forever.

If I were still a Unity employee I would probably have simply fixed the issue and that would be the end of the story, but currently I do not have any way of pushing this fix to HDRP and this bug is likely to affect URP as well. Hopefully Unity devs see this blog post and fix it, or if someone uses HDRP as a reference, make sure to read this.

## The Visual Artifact

![](/assets/img/posts/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/image.png)
**IBL applied to a cube in Unity’s HDRP**

![](/assets/img/posts/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/image-1.png)
**RenderDoc capture of an HDRP frame, mip0 of the pre-convolved LD texture, X+**

![](/assets/img/posts/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/image-2.png)
**RenderDoc capture of an HDRP frame, mip2 of the pre-convolved LD texture, X+**

## Context: BRDF Pre-convolution for Image Based Lighting

There are several methods for implementing image-based lighting in real-time rendering. The approach discussed here is the split-sum approximation introduced by Brian Karis in his SIGGRAPH 2013 course “Real Shading in Unreal Engine 4” [1]. This method relies on precomputed lookup textures (a 2D texture and a cubemap for each microfacet BRDF) to approximate the integral of the rendering equation for the given BRDF.

The approximation splits the integral into two independent parts:

1. **FGD integration**: Precomputes the Fresnel and geometric terms into a 2D lookup texture as a function of the perceptual roughness and the dot product between the view direction and normal (NdotV).
2. **LD prefiltering**: Convolves the environment map with a simplified BRDF lobe, storing the result in a mipmapped cubemap where each mip level corresponds to a different roughness value.

![](/assets/img/posts/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/image-12.png)
**Rendering equation for a microfacet BRDF**

## Preconvolving Efficiently

The FGD texture is static, so it can be generated once independently of how frequently the environment map changes. However, the LD convolution can be performed in real-time by engines.

This may be necessary if you have dynamic reflection probes or a dynamic environment map (with moving clouds, for instance). Thus, you need to perform this convolution each frame and ensure that your signal is completely noise-free for material evaluation.

For LDR textures, it is not too complex to produce noise-free results for the LD convolution within a console-compatible real-time budget, given the relatively low variance of the various samples being combined. However, if you have HDR content (which is more interesting and likely more realistic), you need either a large number of samples or a non-brute-force approach.

![](/assets/img/posts/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/image-4.png)
*Values in the [0.0, 2.0] range*

![](/assets/img/posts/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/image-3.png)
*Values in the [0.0, 48k] range*

Many options are available for reducing the cost and variance. Here is a non-exhaustive list:

- Prefiltering the cubemap into its mip levels
- Pre-generating a sequence with good domain coverage
- Adaptive sample count based on roughness per mip level
- BRDF importance sampling
- Environment map importance sampling
- Multiple importance sampling
- Clamping the input cubemap to more reasonable values (**don’t do it or don’t tell your artists**)

In HDRP, all of these are implemented (except the last one), which makes this convolution “cheap” at runtime. However, our bug is due to a combination of two optimizations.

## An unlucky combination

Here is pseudo-code that summarizes the global steps of the LD convolution:

```hlsl
// The base resolution of the texture
const uint32_t baseRes = 1024;

// For each mip
for (uint mipIdx = 0; mipIdx < mipCount; ++mipIdx)
{
    // Resolution of the current mip
    const uint32_t mipRes = baseRes >> mipIdx;

    // For each face
    for (uint cubeFace = 0; cubeFace < faceCount; ++cubeFace)
    {
        // For each pixel
        for (uint yIdx = 0; yIdx < mipRes; ++yIdx)
        {
            for (uint xIdx = 0; xIdx < mipRes; ++xIdx)
            {
                // Evaluate the view direction
                float3 viewDir = evaluate_view_direction(mipRes, cubeFace, xIdx, yIdx);

                // Generate a local frame
                float3x3 localBasis = get_local_frame(-viewDir);

                // For each sample
                for (uint sampleIdx = 0; sampleIdx < sampleCount; ++sampleIdx)
                {
                    // Generate a 2D random sample
                    float2 u = your_favorite_sequence(sampleIdx, sampleCount);
                    
                    // Importance sample the signal (BRDF, Envmap, MIS, etc)
                    float3 L = importance_sample(...);
                    
                    // Accumulate contribution, normalize, etc.
                    ...
                }
            }
        }
    }
}
```

In this pseudo code, **each pixel uses the exact same sequence**, which introduces bias, correlation and aliasing.

This is mathematically incorrect, but it doesn’t really matter on its own as it is mostly imperceptible in most cases. Remember, we are doing real-time rendering here, bias is often acceptable! On top of that, the split-sum approximation is mathematically biased anyway, so who cares really?

![](/assets/img/posts/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/image-5.png)
*LD term, mip2 on the Z- face, not showing any issues*

The visual issue is caused by this wrongful usage of the sequence combined with this specific code:

```hlsl
// Ref: 'ortho_basis_pixar_r2' from http://marc-b-reynolds.github.io/quaternions/2016/07/06/Orthonormal.html
float3x3 get_local_frame(float3 localZ)
{
    ...
}

// Generate a local frame
float3x3 localBasis = get_local_frame(-viewDir);
```

Don’t get me wrong, this local frame generation is **very useful and stable**, but it has a singularity exactly where we are looking.

As I previously discussed in this blog post on NDF and VNDF GGX importance sampling **[2]**, a local tangent frame is required for these sampling methods.

If the signal is anisotropic, the exact basis definition is important, but if the signal is isotropic (which is the case for the LD convolution), it doesn’t really matter as long as it doesn’t introduce any bias in the integration that would fail at the singularity, which is exactly what happens here.

To understand why this happens, consider an 8×8 grid of pixels around the center of either the X+ or X- face, as shown in the figure below. The pixels in green are on the left half of the face and the pixels in red are on the right half.

![An 8x8 grid of pixels numbered 0 to 63, the left four columns green and the right four red](/assets/img/posts/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/image-13.png)

Now suppose we have an arbitrary sequence of 2D samples used for importance sampling within the [0, 1]² domain, as shown in the figure below.

Going from pixel to pixel while staying on the same side, we have only slight variations in the generated local basis, in practice, the samples are warped very slightly within the domain. However, once we cross the center boundary, one of the tangent axes flips direction. Due to this axis flip, the sample coverage pattern mirrors itself, creating a visible discontinuity at the seam. This is exactly what we observe in the artifact.

![Two scatter plots of sample points in the unit square, the left side green and the right side red, mirrored in u](/assets/img/posts/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/image-16.png)

## Solution(s)
There are multiple ways to fix this. The bad news is that they all increase the variance of the final signal, as we’ll be trading bias/correlation for variance. Here are a few solutions:

- Use a low-discrepancy blue noise function. This would allow you to keep the sample quality high and filterable using a simple gaussian blur **[3]**.
- Introduce a seed per pixel that would use a different part of the sequence that you already have, but this will make the variance skyrocket per pixel as we’d lose the benefit of pre-computed sequences and would be inefficient once you’ve reached the maximum size of your sequence (for the lowest mips for instance).
- Use a local-basis-less VNDF importance sampling method **[2]**.
- For each pixel rotate in a stochastic fashion the local arbitrary basis.
- Accept that ugly line and pretend it doesn’t exist.

Here is how it looks before and after fixing the issue:

![](/assets/img/posts/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/image-2.png)
*X+ face of the LD texture, mip2, **before***

![](/assets/img/posts/sample-decorrelation-in-brdf-preconvolution-for-image-based-lighting/image-7.png)
*X+ face of the LD texture, mip2, **after***

## Conclusion
This bug is a perfect example of how multiple seemingly reasonable optimizations can interact in unexpected ways. Using a pre-generated low-discrepancy sequence is good practice, and using an efficient local frame generation is equally sensible, but combine them and **TADA**, a singularity creates visible artifacts.

In practice, this issue went unnoticed for years because it’s rarely visible in complex production assets with varied environment maps and surface properties. The seam only becomes apparent under very specific conditions: it requires a rough metallic plane, white specular color, with no roughness variation facing exactly the X+ or X- direction to be observable. This explains why it survived so long in production code. Most real-world scenes have varied surface orientations and roughness maps that mask the artifact.

For those implementing their own IBL systems or using HDRP as a reference, the key takeaway is simple: ensure sample or basis decorrelation across pixels, whether through blue noise offsets, per-pixel seeds, or using sampling methods that don’t require constructing a local basis. The choice depends on your performance budget and quality requirements.

If you’re a Unity developer reading this, the fix should be straightforward to implement. I’d be happy to see this resolved in future HDRP versions.

**References**

[1] Karis, B. (2013). Real Shading in Unreal Engine 4. SIGGRAPH 2013 Course: Physically Based Shading in Theory and Practice. <https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf>

[2] [VNDF importance sampling for an isotropic Smith-GGX distribution](/2024/04/15/vndf-importance-sampling-an-isotropic-distribution/)

[3] Heitz, E., & Belcour, L. (2019). A Low-Discrepancy Sampler that Distributes Monte Carlo Errors as a Blue Noise in Screen Space. ACM SIGGRAPH Talks.
