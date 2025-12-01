---
title: "Slang for Cross-Platform Ray Tracing: Results from my experiments"
date: 2025-11-25T09:00:00Z
draft: false
categories: [graphics, rendering, experiments]
tags: [slang, shaders, portability]
---
 
## TL;DR

**Can a single shader language really deliver native performance across DXR and Vulkan?** I integrated Slang into [ChameleonRT](https://github.com/Twinklebear/ChameleonRT) to find out. Across five test scenes, Slang matched or exceeded hand-written HLSL/GLSL in all but one anomalous case—with an average 30% performance gain on DXR. It's early, but the promise of true shader portability is looking real.

{{< image-carousel "/images/rungholt_comparison.png,/images/sanmiguel_comparison.png" "Rungholt: Visual comparison across backends|San Miguel: Visual comparison across backends" >}}

## Background and Motivation

What drew me to Slang was its promise of true shader portability. I’ve always been interested in technologies that transcend their usual boundaries—like USD for scene portability or MaterialX for material portability. In the same vein, Slang offered a chance to achieve a similar kind of portability for graphics shaders, something that felt like a natural evolution beyond something like SYCL.

Initially, I experimented with Slang-GFX and its raytracing pipeline examples, trying to build a simple app around it. But after about a week of tinkering, it became clear that using Slang's full framework was more cumbersome than needed for my goal. It also turned out that Slang-GFX was on its way to being replaced by Slang-RHI, which was still under development. Instead of wrestling with an abstraction layer, I realized I could just integrate Slang purely as the shading language inside [ChameleonRT](https://github.com/Twinklebear/ChameleonRT).

ChameleonRT was the perfect playground because it already supported multiple backends like DXR and Vulkan, making it easier to do a side-by-side comparison. This way, I could focus on what Slang does best (shader portability) without getting bogged down in re-architecting my entire pipeline.

## Aligning DXR and Vulkan for True Shader Portability

When I began integrating Slang, it quickly became clear that shader portability wasn't just a language problem. It was architectural. Both DXR and Vulkan executed the same ray-tracing algorithms, but the way each pipeline exposed geometry data to shaders was completely different.

In DXR, per-geometry buffers (vertex, index, normal, UV) were bound through local root signatures. Each geometry wrote its GPU virtual addresses into the Shader Binding Table, and the Closest Hit shader accessed them through space 1 bindings:

```hlsl
StructuredBuffer<float3> vertices : register(t0, space1);
StructuredBuffer<uint3>  indices  : register(t1, space1);
cbuffer MeshData : register(b0, space1) { uint material_id; };
```

In Vulkan, the same data took a completely different route. Each geometry's buffers were written into the SBT via a shaderRecordEXT block, and the shader dereferenced raw GPU addresses with the buffer_reference extension:

```glsl
layout(shaderRecordEXT, std430) buffer SBT {
	VertexBuffer verts;
	IndexBuffer indices;
	uint material_id;
};
```

The logic was identical, but the data-access semantics were not. DXR injected resources through its root-signature indirection, while Vulkan shaders manually dereferenced 64-bit pointers. Because of this, a portable shader could never be "clean" across both backends.

I flattened and re-organized Vulkan's descriptor layout to mirror DXR's single-heap model. All resources (scene, geometry, and textures) now live under Set 0, creating a one-to-one mapping with DXR's space 0.

One deliberate decision at this stage was to avoid introducing Slang's Reflection API. While reflections could have automated parts of the binding translation, my goal was to make the smallest, most controlled set of changes required to reach functional portability, rather than add another variable to an already shifting system.

**Table: Unified Resource Binding Layout (DXR ↔ Vulkan)**

| Resource | Vulkan (Set 0 Binding) | DXR (Register / Space 0) |
|---|---:|---:|
| TLAS | 0 | t0 |
| Render Target | 1 | u0 |
| Accum Buffer | 2 | u1 |
| View Params | 3 | b0 |
| Materials | 4 | t1 |
| Lights | 5 | t2 |
| Global Buffers (vertices, indices, etc.) | 10–14 | t10–t14 |
| Textures | 30+ | t30+ |
| Sampler (shared) | s0 | s0 |

This flattened design removed Vulkan’s second descriptor set entirely, aligned resource numbering across both APIs, and made the shaders nearly identical apart from syntax.

As a result, DXR and Vulkan now share the same logical bindings, enabling the same shaders to run across both backends with minimal changes. The Shader Binding Table became simpler and smaller, since it no longer stored per-geometry GPU addresses—just offsets into global buffers. While some performance trade-offs were expected when moving away from fine-grained geometry bindings, it was a deliberate compromise to unlock portability.

## Integrating Slang into ChameleonRT

With the descriptor alignment in place, integrating Slang into ChameleonRT was relatively straightforward. Slang's syntax is close to HLSL, so porting existing HLSL shaders into Slang was essentially a direct translation. The main changes were converting HLSL-specific register bindings to Slang's more portable syntax, and ensuring the shader entry points matched what both DXR and Vulkan backends expected.

After replacing the original shader code with Slang, I confirmed that the visual results were identical. I validated this across a range of scenes, from simple test cases to complex geometry. Slang-generated shaders matched the original HLSL/GLSL output perfectly. The entire integration took about two days, including validation.

The language switch was seamless, and I now had a unified shader codebase compiling to both DXR and Vulkan without functional differences.

## Benchmarking and Early Findings

### The Test Platform: ChameleonRT

ChameleonRT is a minimal path tracer designed for comparing hardware ray tracing APIs. Understanding its architecture helps contextualize the benchmark results:

- **Iterative Path Tracing**: All path bounces are handled in a single `RayGen` shader invocation via a loop. The `ClosestHit` shader returns hit data.
- **Camera Model**: Pinhole camera with configurable samples per pixel.
- **Material Model**: Disney BRDF supporting base color, metallic, roughness, specular, anisotropy, sheen, clearcoat, IOR, and transmission parameters.
- **Lighting**: A single quad area light per scene, sampled using Multiple Importance Sampling (MIS) that combines light sampling and BRDF sampling strategies.
- **No Any-Hit Shaders**: All geometry is fully opaque.
- **Path Termination**: Maximum depth of 5 bounces, with Russian roulette applied after bounce 3.

### Hardware

**Hardware / Drivers**

- GPU: NVIDIA RTX 3070
- Driver: 566.03
- OS: Windows 10 SDK 10.0.26100.0

### Platform specifics

| Platform |
|---|
| **DXR**: Shader Model `lib_6_6` • DXC (Win SDK 10.0.26100.0) with `-O3` |
| **Vulkan**: Target: SPIR-V 1.5 • Compiler: glslc (Vulkan SDK 1.4.321.1) with `-O --target-env=vulkan1.2` |
| **Slang**: Version: v2025.12.1 • Profiles: `lib_6_6` (DXIL), `spirv_1_5` (SPIR-V) • Optimization: `SLANG_OPTIMIZATION_LEVEL_HIGH` • SlangPassThrough set with the same DXC and GLSLC |

### Timing Methodology

All render times reported are **GPU-only execution time** measured using D3D12 timestamp queries (DXR) or Vulkan timestamp queries (Vulkan). These timestamps bracket only the `DispatchRays` call, excluding:

- Shader compilation (performed at build time for native HLSL/GLSL, at application startup for Slang)
- CPU overhead and command buffer recording
- Memory transfers and readback operations

This ensures we're measuring pure ray tracing compute performance, not pipeline setup or I/O.

**Scenes Tested** 
( 1920×1080, 8 spp, 10 accumulation frames — each scene run 5× per backend)

- Cornell Box
- Rungholt
- San Miguel
- Breakfast Room
- Salle de Bain

*Test scenes courtesy of [Morgan McGuire's Computer Graphics Archive](https://casual-effects.com/g3d/data10/index.html)*

### Results
| Backend | Scene | Baseline (ms) | Slang (ms) | Δ (%) | Faster |
|---|---|---:|---:|---:|---:|
| DXR | Cornell Box | 13.99 | 8.19 | -41 | Slang |
| DXR | Rungholt | 25.50 | 17.81 | -30 | Slang |
| DXR | San Miguel | 54.82 | 47.23 | -14 | Slang |
| DXR | Breakfast Room | 34.74 | 22.02 | -37 | Slang |
| DXR | Salle de Bain | 35.01 | 26.02 | -26 | Slang |
| Vulkan | Cornell Box | 5.87 | 4.84 | -18 | Slang |
| Vulkan | Rungholt | 13.97 | 12.78 | -9 | Slang |
| Vulkan | San Miguel | 43.79 | 41.96 | -4 | Slang |
| Vulkan | Breakfast Room | 16.02 | 15.25 | -5 | Slang |
| Vulkan | Salle de Bain | 14.98 | **19.36*** | +29 | GLSL (anomaly) |

*Note: The Vulkan Salle de Bain data included two anomalous runs (13.26, 27.09, 13.43, 19.91, 23.10 ms). Ignoring the >20 ms outliers, Slang again edges GLSL.

**Performance Summary:**
- DXR: Slang **29.6% faster** on average
- Vulkan: Slang **1.4% faster** (skewed by Salle de Bain outlier)


## Conclusion and Next Steps

This exploration focused on a contained but representative problem: testing Slang across DXR and Vulkan in a unified environment. Within this scope (single-light scenes, fixed sampling, and straightforward materials), Slang integrated cleanly, compiled reliably, and produced results at least as efficient as hand-authored HLSL/GLSL.

While these results are promising, they represent only a starting point. These scenes do not yet cover advanced lighting, denoising, or adaptive sampling. Understanding Slang's behavior under more complex workloads will be the next step.

**Next steps:**

- **Performance attribution**: Profile the compiler output to identify where the 30% DXR performance gain comes from and validate reproducibility
- **Cross-hardware validation**: Test on other GPUs to establish whether these performance characteristics are universal or vendor-specific
- **Metal portability**: Validate that the same cross-platform shader approach translates cleanly to Apple's Metal API, completing the portability story across all major platforms

I feel confident enough to continue improving my fork of ChameleonRT with Slang as the sole shading language. I no longer need to maintain parallel HLSL paths and plan to evolve the project entirely within the Slang ecosystem going forward.

## Acknowledgments

Thanks to [Christoph Peters](https://momentsingraphics.de/) for detailed technical feedback on GPU performance analysis methodology.

