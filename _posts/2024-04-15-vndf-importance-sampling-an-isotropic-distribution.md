---
layout: post
title: "VNDF importance sampling for an isotropic Smith-GGX distribution"
date: 2024-04-15T13:59:17+00:00
original_url: https://auzaiffe.wordpress.com/2024/04/15/vndf-importance-sampling-an-isotropic-distribution/
---

In this blog post, you will find an implementation for importance sampling a VNDF (GGX-Smith) isotropic distribution that is 15% faster than the current state of the art and doesn’t require building a local basis.

Here is the HLSL implementation:

```cpp
float3 sample_vndf_isotropic(float2 u, float3 wi, float alpha, float3 n)
{
    // decompose the floattor in parallel and perpendicular components
    float3 wi_z = -n * dot(wi, n);
    float3 wi_xy = wi + wi_z;

    // warp to the hemisphere configuration
    float3 wiStd = -normalize(alpha * wi_xy + wi_z);

    // sample a spherical cap in (-wiStd.z, 1]
    float wiStd_z = dot(wiStd, n);
    float z = 1.0 - u.y * (1.0 + wiStd_z);
    float sinTheta = sqrt(saturate(1.0f - z * z));
    float phi = TWO_PI * u.x - PI;
    float x = sinTheta * cos(phi);
    float y = sinTheta * sin(phi);
    float3 cStd = float3(x, y, z);

    // reflect sample to align with normal
    float3 up = float3(0, 0, 1.000001); // Used for the singularity
    float3 wr = n + up;
    float3 c = dot(wr, cStd) * wr / wr.z - cStd;

    // compute halfway direction as standard normal
    float3 wmStd = c + wiStd;
    float3 wmStd_z = n * dot(n, wmStd);
    float3 wmStd_xy = wmStd_z - wmStd;
    
    // return final normal
    return normalize(alpha * wmStd_xy + wmStd_z);
}
```

Where ***wi*** is the view vector in world space, ***n*** the normal in world space, ***alpha*** the isotropic roughness and ***u*** a pair of random values (usually they are stratified and dithered).  
  
You can find the GLSL implementation on **[Jonathan’s github](https://gist.github.com/jdupuy/4c6e782b62c92b9cb3d13fbb0a5bd7a0#file-samplevndf_ggx-cpp-L51).**  
  
For what’s remaining, we’ll give more details about the speedup.

Last year, with my colleague Jonathan “**Omar**” Dupuy, we published a paper at HPG 2023 named “Sampling Visible GGX Normals with Spherical Caps”. The article provides a new way to approach the visible normal distribution function (**VNDF**) importance sampling routine using *spherical caps*. While offering a new perspective into how to approach the problem, it also provides a substantial speedup (45% on average for the sampling routine itself) w/r to the previous state of the art by Heitz.

![](/assets/img/posts/vndf-importance-sampling-an-isotropic-distribution/image.png)
*Spherical cap distribution of reflected rays.*

![](/assets/img/posts/vndf-importance-sampling-an-isotropic-distribution/image-1.png)
*Reflection of parallel rays.*

The sampling algorithm is as follows:

- Convert the view vector to a local space where the normal is (0, 0, 1), this requires having or building a world to local matrix (**WorldToLocal**).
- Stretch the local space view vector using the anisotropic roughness (*ax* and *ay*) properties of the surface.
- Use the sampling routine (Heitz or Ours), this produces a normal according to the visible normal distribution function.
- Stretch back the sampled local space normal using the anisotropic roughness (*ax* and *ay*) properties of the surface.
- Convert back the sampled light direction into world space using the **LocalToWorld** matrix.
- Reflect (in world space) the input view vector w/r to the sampled microfacet normal to produce a local space light direction.

Once we have generated this direction, we shall use it, for instance, to trace the following section of the light path.

The questioning we had was: Is there a way to have a specialized routine for the isotropic case (given that it is the most frequent one) that would make either the sampling process faster or avoiding to build a local basis (or both?).

## Building the local basis
Currently, when you are doing VNDF importance sampling for the isotropic case (at least for games), you do not have a tangent space basis available because the tangent itself is not available for the pixel/texel we are trying to shade (for GBuffer packing constraints, or simply vertex to fragment payload size). This usually means, we’ll try to build a local basis on the fly and transpose it for the last step of the sampling routine. A common way of building this basis is the following:

- If the normal and view vector are not aligned, we’ll build a local basis using these two vectors.
- In the other case we’ll generate a basis using the frisvad or reynold’s routines using the world space normal. <http://marc-b-reynolds.github.io/quaternions/2016/07/06/Orthonormal.html>

In practice this works fine, but requires branching and storing 9 FP32 values in the VGPRs during the whole routine (plus the cost of the generation).

The insight we had is: For the isotropic case, we can see the **WorldToLocal** and **LocalToWorld** transformations as reflect operations w/r to the half vector between the normal in world space and normal in local space (Z Up).  
  
Both the usual technique and our proposal have a singularity (in practice, we don’t observe issues in either cases):

- For the state of the art approach, it happens when N and V are aligned.
- For our approach it happens when the normal vector is (0, 0, -1).

## Peformance
We needed to evaluate the potential speedup of our method. To do so, we profiled both the state of the art approach and ours. The benchmark consists of a fragment shader generating 1024 light directions while varying for each generation the random numbers, the view vector and the normal vector to avoid the compiler caching the transformation matrix or simplifying evaluations that are variant.

When comparing to the previous state of the art method, we have an average speedup of **15%**, which in our opinion is interesting.

## Appendix:
## PDF of an isotropic sample
When combining VNDF importance sampling with other techniques such as multiple importance sampling (MIS) or resampled importance sampling (RIS), it is important to have the PDF of the sample that was generated beyond it’s actual evaluation. This is not new data, but we thought it would be interesting to have it with the importance sampling routine:

```cpp
float pdf_vndf_isotropic(float3 wo, float3 wi, float alpha, float3 n)
{
    float alphaSquare = alpha * alpha;
    float3 wm = normalize(wo + wi);
    float zm = dot(wm, n);
    float zi = dot(wi, n);
    float nrm = rsqrt((zi * zi) * (1.0f - alphaSquare) + alphaSquare);
    float sigmaStd = (zi * nrm) * 0.5f + 0.5f;
    float sigmaI = sigmaStd / nrm;
    float nrmN = (zm * zm) * (alphaSquare - 1.0f) + 1.0f;
    return alphaSquare / (M_PI * 4.0f * nrmN * nrmN * sigmaI);
}
```
