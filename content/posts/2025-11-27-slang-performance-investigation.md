---
title: "Why Is Slang ~30% Faster on DXR?"
date: 2025-11-27T09:00:00Z
draft: false
categories: [graphics, rendering, experiments]
tags: [slang, shaders, performance, optimization]
---

After integrating Slang into ChameleonRT and seeing a ~30% speedup on DXR, I needed to know: was this real, or had I made a mistake in my baseline? This post documents that investigation.

## TL;DR

**Why was Slang ~30% faster?** After eliminating compiler flags and configuration differences, I traced the improvement to the generated code. Slang's frontend applies optimizations like common subexpression elimination, constant hoisting, redundant load removal before emitting HLSL. DXC then compiles this pre-optimized code. The native HLSL→DXC path doesn't perform these transformations, leaving performance on the table. NSight profiling across multiple scenes confirmed the speedup was real and consistent.

## The Unexpected Performance Gap

In my [previous post](/posts/2025-11-25-slang-experiments/), I integrated Slang into ChameleonRT and benchmarked it against hand-written HLSL and GLSL shaders. The results were surprising: Slang showed an average **29.6% performance improvement on DXR** and modest gains on Vulkan. While I expected Slang to match native performance, I didn't expect it to significantly exceed it.

These numbers were hard to believe. The results suggested that either I had made a mistake in my baseline implementation, or Slang's compiler was doing something fundamentally different (and better) than what I'd written by hand.

I needed to understand what was happening. Was this a measurement artifact? A difference in compiler flags? Or was Slang genuinely generating more efficient code? I started investigating, focusing primarily on the DXR/HLSL path where the performance gap was most pronounced.

## Ensuring an Apples-to-Apples Comparison

The first step was to eliminate every possible variable that might explain the performance difference. If I had inadvertently given Slang an unfair advantage through different compiler settings or optimizations, that would invalidate the entire comparison.

### Compiler Alignment

I started by verifying that Slang was using the exact same downstream compilers as my baseline builds. While Slang can generate DXIL and SPIR-V directly through its backend, I wanted to ensure a controlled comparison by forcing it to use the same DXC and glslc that my native builds relied on.

I configured Slang to pass through to specific compiler executables:

```cpp
// Force Slang to use the same DXC as native HLSL builds
globalSession->setDownstreamCompilerPath(
    SLANG_PASS_THROUGH_DXC,
    "<Windows SDK path>/dxc.exe"
);

// Force Slang to use the same glslc as native GLSL builds  
globalSession->setDownstreamCompilerPath(
    SLANG_PASS_THROUGH_GLSLANG,
    "<Vulkan SDK path>/glslc.exe"
);
```

This configuration ensured that Slang would generate HLSL or GLSL as an intermediate step, then invoke the exact same compilers used by my baseline builds to produce the final bytecode.

### Optimization Flag Parity

Next, I aligned optimization flags across all three paths: `-O3` for DXC, `-O --target-env=vulkan1.2` for glslc, and `SLANG_OPTIMIZATION_LEVEL_HIGH` for Slang. I verified that all three were targeting the same shader models (DXR: `lib_6_6`, Vulkan: SPIR-V 1.5). I also discovered that Slang automatically enables `-enable-16bit-types` for Shader Model 6.2+, so I added that to my native build as well. With identical compilers, optimization levels, and flags, I expected the performance gap to disappear. It didn't.

## Digging Deeper: Comparing the Generated Code

If compiler flags weren't the issue, then the difference had to be in the code itself. I needed to see what both compilers were actually generating. I started by disassembling the DXIL bytecode from both paths to count instructions and identify differences.

### Instruction Count Analysis

I'll be honest. Disassembling DXIL and analyzing instruction counts was well outside my usual comfort zone, but the performance gap was too significant to ignore, so I rolled up my sleeves and learned just enough to be dangerous.

Using `dxc.exe -dumpbin` on both the native HLSL and Slang-generated DXIL files, I wrote some simple scripts to count specific instruction types. My assumption was that if Slang's backend was truly generating more efficient code, I should see fewer instructions in key categories.

The numbers confirmed it:

| Metric | HLSL (DXC) | Slang (DXC)| Improvement |
|--------|----------:|-------------:|------------:|
| Function calls (`call`) | 512 | 411 | **-20%** |
| Multiply ops (`fmul`) | 966 | 654 | **-32%** |
| Add ops (`fadd`) | 384 | 260 | **-32%** |
| Division ops (`fdiv`) | 182 | 133 | **-27%** |
| Subtract ops (`fsub`) | 215 | 152 | **-29%** |
| Conditional branches | 101 | 82 | **-19%** |
| Basic blocks | 191 | ~146 | **-24%** |

Slang was generating 20-32% fewer instructions across nearly every category. Same source logic, dramatically different compiled output. But *why*? To understand that, I needed to look at something more readable than raw DXIL assembly.

### The Intermediate HLSL Revelation

This is where things got interesting. Slang has a feature that lets you dump the intermediate HLSL it generates before passing it to DXC. This turned out to be far more illuminating than staring at DXIL disassembly, I could actually read and understand what was happening.

I dumped the Slang→HLSL output and compared it side-by-side with my hand-written code. Here's what I found, using the Disney BRDF evaluation as an example (one of the hottest code paths in the ray tracer):

**Hand-written HLSL:**
```hlsl
float3 disney_microfacet_isotropic(in const DisneyMaterial mat, in const float3 n,
    in const float3 w_o, in const float3 w_i)
{
    float3 w_h = normalize(w_i + w_o);
    float lum = luminance(mat.base_color);
    float3 tint = lum > 0.f ? mat.base_color / lum : float3(1, 1, 1);
    float3 spec = lerp(mat.specular * 0.08 * lerp(float3(1, 1, 1), tint, 
                       mat.specular_tint), mat.base_color, mat.metallic);

    float alpha = max(0.001, mat.roughness * mat.roughness);
    float d = gtr_2(dot(n, w_h), alpha);
    float3 f = lerp(spec, float3(1, 1, 1), schlick_weight(dot(w_i, w_h)));
    float g = smith_shadowing_ggx(dot(n, w_i), alpha) * 
              smith_shadowing_ggx(dot(n, w_o), alpha);
    return d * f * g;
}
```

**Slang's generated HLSL:**
```hlsl
float3 disney_microfacet_isotropic_0(DisneyMaterial_0 mat_5, float3 n_12, 
                                      float3 w_o_10, float3 w_i_11)
{
    float3 w_h_7 = normalize(w_i_11 + w_o_10);
    float lum_1 = luminance_0(mat_5.base_color_1);
    float3 tint_1;
    if(lum_1 > 0.0f) {
        tint_1 = mat_5.base_color_1 / lum_1;
    } else {
        tint_1 = float3(1.0f, 1.0f, 1.0f);
    }
    
    // This constant appears once, then gets reused
    float3 _S96 = float3(1.0f, 1.0f, 1.0f);
    float3 spec_0 = lerp(mat_5.specular_1 * 0.08f * lerp(_S96, tint_1, 
                         (float3)mat_5.specular_tint_1), 
                         mat_5.base_color_1, (float3)mat_5.metallic_1);

    // Field access happens once, not twice
    float _S97 = mat_5.roughness_1;
    float alpha_10 = max(0.001f, _S97 * _S97);
    float d_7 = gtr_2_0(dot(n_12, w_h_7), alpha_10);
    
    // Function call result gets cached
    float _S98 = schlick_weight_0(dot(w_i_11, w_h_7));
    float3 f_3 = lerp(spec_0, _S96, (float3)_S98);
    
    // Both function results cached in temporaries
    float _S99 = smith_shadowing_ggx_0(dot(n_12, w_i_11), alpha_10);
    float _S100 = smith_shadowing_ggx_0(dot(n_12, w_o_10), alpha_10);
    float g_3 = _S99 * _S100;
    
    return d_7 * f_3 * g_3;
}
```

At first glance, Slang's version looks more verbose with all those numbered temporaries (`_S96`, `_S97`, etc.). As I tried to understand why, I realized what was happening. These weren't arbitrary additions. They were deliberate optimizations. I had to look up what some of these techniques were called. Techniques like "common subexpression elimination" and "constant hoisting" weren't part of my daily vocabulary, but the principle was clear: Slang's frontend was recognizing optimization opportunities that I hadn't thought about when writing the code by hand, and DXC wasn't catching when compiling my HLSL.

The instruction count reductions mapped directly to these optimizations. The native HLSL→DXC path compiled my code as written; Slang restructured it first. On Vulkan, improvements were more modest (5-10%)—glslc likely already applies similar optimizations.

## Validating with NSight GPU Profiling

The instruction count analysis was compelling, but I wanted GPU-level validation. I captured NSight traces across multiple scenes to see if the profiler data matched the benchmark results.

### What NSight Showed

| Scene | Version | Registers | Warp Occupancy | Frame Time |
|-------|---------|----------:|---------------:|-----------:|
| Cornell Box | Native HLSL | 118 | 29.2% | ~17 ms |
| Cornell Box | Slang | 107 | 23.8% | ~12 ms |
| San Miguel | Native HLSL | 118 | 29.5% | ~67 ms |
| San Miguel | Slang | 107 | 29.0% | ~55 ms |
| Rungholt | Native HLSL | 118 | 27.3% | ~36 ms |
| Rungholt | Slang | 107 | 28.9% | ~23 ms |

*Frame times are approximate and include NSight overhead. The relative differences are what matter.*

### Observations

**Register counts are consistent.** The hottest shader uses 118 registers with native HLSL and 107 with Slang—an 11-register reduction that holds across all scenes.

**The performance improvement holds across scene complexity.** Frame time improvements range from ~18% (San Miguel) to ~36% (Rungholt), broadly consistent with the benchmark results from the previous post.

### The Limits of My Analysis

I can see the *correlation* between Slang's optimizations and the speedup, but establishing precise *causation* requires deeper expertise than I have. For anyone who wants to dig deeper, I'd recommend Christoph Peters' [work on register pressure](https://momentsingraphics.de/GPUPolynomialRoots.html).

{{< image-carousel "/images/DXR-No-Slang.png,/images/DXR-With-Slang.png,/images/SanMiguel-NoSlang.png,/images/SanMiguel-WithSlang.png,/images/RungHolt-NoSlang.png,/images/Rungholt-WithSlang.png" "Cornell Box: Native HLSL|Cornell Box: Slang|San Miguel: Native HLSL|San Miguel: Slang|Rungholt: Native HLSL|Rungholt: Slang" >}}

## Conclusions

The performance gap was real. After eliminating compiler flags and configuration as factors, the difference came down to code. Slang's frontend restructures shaders before downstream compilers ever see them, transformations those compilers probably aren't performing on their own.

### What This Means in Practice

For teams evaluating shader languages:

- **The speedup appears real and consistent** across the scenes I tested, though hardware and workload dependent.
- **Cleaner shader code** becomes possible when the compiler handles micro-optimizations.
- **The barrier to entry lowers** for shader authors who aren't HLSL optimization experts.

### Limitations

This analysis has important caveats:

- **Single GPU tested.** All benchmarks ran on RTX 3070. Results may differ on other architectures.
- **ChameleonRT-specific.** The workload is an iterative path tracer with specific characteristics (Disney BRDF, MIS, area lights). Production engines have different shader patterns.
- **Correlation, not causation.** I've shown *what* changed (instruction counts, registers) and *that* performance improved, but I haven't proven the precise causal chain. That would require deeper profiling beyond my current expertise.
- **DXR focus.** The Vulkan improvements were modest (5-10%). This investigation focused on the DXR path where the gap was largest.

### Final Thoughts

For my goal of building a cross-platform ray tracing testbed—Slang delivers meaningful performance gains with less manual optimization effort. Whether these results generalize to production engines with different shader patterns is a question I can't answer definitively. I'd encourage anyone interested to run their own benchmarks.

## Acknowledgments

Special thanks to Christoph Peters for detailed technical feedback on an earlier draft of this post. His expertise in GPU performance analysis helped identify gaps in my methodology and sharpen the presentation of results. Any remaining errors are my own.
