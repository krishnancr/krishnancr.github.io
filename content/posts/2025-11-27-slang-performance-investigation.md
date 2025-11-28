---
title: "Why Is Slang ~30% Faster on DXR?"
date: 2025-11-27T09:00:00Z
draft: false
categories: [graphics, rendering, experiments]
tags: [slang, shaders, performance, optimization]
---

Shader systems are becoming a core architectural layer in modern rendering pipelines. After integrating Slang into ChameleonRT, the next natural question was identifying the root cause for performance gain. This post focuses on understanding the surprising performance differences I observed on DXR and why those differences matter for cross-platform engine design.

## TL;DR

**Why was Slang ~30% faster?** After eliminating compiler flags and configuration differences, I found the answer in the generated code. Slang's frontend applies optimizations before emitting HLSL. DXC then compiles this pre-optimized code. Native HLSL→DXC does not seem to do this, leaving performance on the table. NSight GPU profiling validated everything: 32% fewer instructions translated to 43% less FMA pipe usage. The speedup was real.

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

Next, I aligned optimization flags across all three paths: `-O3` for DXC, `-O --target-env=vulkan1.2` for glslc, and `SLANG_OPTIMIZATION_LEVEL_HIGH` for Slang. I verified that all three were targeting the same shader models (DXR: `lib_6_6`, Vulkan: SPIR-V 1.5). With identical compilers, optimization levels, and target specifications, I expected the performance gap to disappear. It didn't.

### The First "Aha" Moment: `-enable-16bit-types`

Something still wasn't matching up. I started digging through Slang's codebase and documentation to understand what compilation flags it was setting under the hood. That's when I found it. Buried in the compiler's DXIL backend code, Slang automatically enables 16-bit type support when targeting Shader Model 6.2 or higher.

This flag, `-enable-16bit-types`, enables native 8-bit and 16-bit type alignment. Without it, DXC conservatively pads smaller types to 32-bit boundaries, which wastes memory bandwidth and degrades cache efficiency. My native HLSL build wasn't using this flag at all.

This seemed like the smoking gun. If this was the source of the 30% performance difference, then the comparison wasn't apples-to-apples. Slang was simply using a better default configuration. I immediately updated my native HLSL build:

```cmake
COMPILE_OPTIONS -O3 -enable-16bit-types
```

Rebuilt. Re-ran the benchmarks. **The performance gap remained.** The native HLSL build was still 30% slower than Slang, even with identical compiler flags. Whatever Slang was doing, it went deeper than just enabling modern type support.

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

**My hand-written HLSL:**
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

At first glance, Slang's version looks more verbose with all those numbered temporaries (`_S96`, `_S97`, etc.). But as I studied it, I realized what was happening. These weren't arbitrary additions. They were deliberate optimizations. I had to look up what some of these techniques were called. Techniques like "common subexpression elimination" and "constant hoisting" weren't part of my daily vocabulary, but the principle was clear: Slang's frontend was recognizing optimization opportunities that I hadn't thought about when writing the code by hand, and DXC wasn't catching when compiling my HLSL.

The beauty of this approach is that these optimizations happen, *before* Slang even emits HLSL. When DXC compiles this pre-optimized code, it's already in a form that's easier to generate efficient machine code from.

### Where the Performance Actually Comes From

The instruction count reductions mapped directly to the optimizations visible in the generated code. Aggressive inlining, constant hoisting, CSE, and other optimizations all contributed to the reduction of arithmetic operations. The native HLSL→DXC path compiled my code as written; Slang's pipeline restructured it into a more efficient form before handing it to DXC. This explained the 30% DXR gap was not a measurement error or compiler flags, but optimization opportunities that required low level HLSL expertise. On Vulkan, improvements were more modest (5-10%) since glslc was likely already applying similar optimizations.

## Validating the Theory with Real GPU Metrics

The instruction count analysis was compelling, but I needed GPU-level validation. I captured NSight traces comparing native HLSL against Slang. If the instruction reductions were real, they should show up in execution unit utilization—fewer arithmetic ops should mean less FMA pipe pressure, fewer calls should reduce XU usage.

The NSight data validated everything I'd found in the DXIL analysis:

| GPU Metric | Native HLSL | Slang | Analysis |
|------------|------------:|------:|----------|
| **L2 Cache Hit Rate** | 70.4% | 74.8% | **+6.2%** (Better memory efficiency) |
| **SM XU Pipe Throughput** | 36.3% | 15.3% | **-58%** (Far fewer address calculations) |
| **SM FMA Pipe Throughput** | 31.5% | 18.1% | **-43%** (Matches the 32% arithmetic reduction) |
| **Compute Warp Latency** | 538,420 cycles | 250,988 cycles | **-53%** (Warps retire much faster) |
| **MIO Throttle Stalls** | 0.3% | 0.1% | **-67%** (Less memory I/O contention) |

The correlation was remarkable. Every prediction from the instruction analysis appeared in the hardware metrics. The 32% fewer arithmetic ops matched the 43% FMA pipe reduction. Even L2 cache hit rates improved, likely from more compact code leaving more room for data.

Vulkan showed a similar but more modest pattern, consistent with the 5-10% performance gain.

These hardware-level measurements gave me the confidence I needed. The performance improvements weren't just artifacts of how I was counting instructions or quirks of the compiler output. They were real, measurable differences in how the GPU was executing the code.

{{< image-carousel "/images/DXR-No-Slang.png,/images/DXR-With-Slang.png" "DXR Without Slang|DXR With Slang" >}}

## Conclusions

The investigation made the source of the performance difference clear: Slang’s multi-stage compilation pipeline restructures shader code before it ever reaches DXC or glslc. By applying canonical compiler optimizations early Slang hands the downstream compilers a more optimized starting point. The direct HLSL→DXC path simply never performs this level of optimizations. That answers the core technical question I set out to understand. For the purposes of building a cross-platform ray tracing testbed, the explanation is complete and actionable.

### What This Means for Teams and Engine Builders

The broader implication is that shader performance no longer has to be just a function of expert level handwritten code, it could be a function of the compiler you choose. For teams building cross-API engines:

- **Automatic performance wins.** A capable shading language can surface meaningful gains without low-level shader micro-tuning.
- **Cleaner, easier-to-maintain shaders.** Engineers can write clear code without worrying about low-level math restructuring or optimization tricks.
- **Lower barrier to entry for shader authors.** Contributors don't need deep HLSL/GLSL micro-optimization expertise to produce performant code.
- **Faster iteration and safer refactoring.** The compiler absorbs most optimization concerns, letting teams modify or extend shader logic without fear of hidden regressions.

### Final Takeaway

This work reinforced how much leverage a high-level shading language with a modern optimizing frontend can provide. Slang raises the floor on shader performance by automating a category of improvements that are tedious and error-prone to perform manually, and it does so without complicating authoring or portability.

And notably, all of this comes before tapping into Slang’s differentiable programming capabilities—the feature that positions it for the next wave of neural and hybrid rendering techniques. Seeing this level of benefit purely from its core compilation path is a strong signal about the direction shader languages are heading and the kinds of systems worth building on.


