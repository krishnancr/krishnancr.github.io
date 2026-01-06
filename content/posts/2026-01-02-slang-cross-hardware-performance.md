---
title: "Slang Cross-Vendor Benchmarks: Performance Wins and Trade-offs"
date: 2026-01-02T09:00:00Z
draft: true
categories: [graphics, rendering, experiments]
tags: [slang, shaders, performance, benchmarking, multi-vendor]
---

## TL;DR

After seeing 30% DXR gains on my RTX 3070, the next question was obvious: does this hold across different hardware? I was able to test Slang across four different GPUs from three vendors. **The results tell a nuanced story: cross-vendor portability works reliably, performance optimization is contextual.** Slang delivered consistent visual results everywhere, with performance ranging from significant gains to regressions in specific cases. The takeaway? [Write Shaders Once, Run Anywhere](https://github.com/shader-slang/slang?tab=readme-ov-file#write-shaders-once-run-anywhere): that promise holds for standard graphics APIs. Whether you're faster depends on your hardware.

{{< image-carousel "/images/slang_perf_comparison.png" "Slang Performance Comparison" >}}

## Background

This is the third post in my Slang exploration series:
- **[Post #1](/posts/2025-11-25-slang-experiments/)**: Can Slang deliver portability? (Yes, with visual parity)
- **[Post #2](/posts/2025-11-27-slang-performance-investigation/)**: Why was NVIDIA DXR 30% faster? (Compiler frontend optimizations)
- **Post #3 (this post)**: Does this generalize across hardware?

The unexpected performance gains from Post #2 raised an important question: were those results specific to my RTX 3070, or would they hold across different GPUs and vendors? To find out, I needed to test on hardware I didn't own.

## Methodology

### Hardware Tested

I was able to benchmark across four GPUs:

| GPU | Vendor | Driver Version | OS |
|-----|--------|----------------|-----|
| NVIDIA RTX 3070 | NVIDIA | 566.03 | Windows 10 |
| NVIDIA RTX 4090 | NVIDIA | 561.09 | Windows 10 |
| Intel B580 (12GB) | Intel | 32.0.101.8247 | Windows 10 |
| AMD Radeon RX 7900 XT | AMD | 25.20.29.09-251203a | Windows 10 |

### Test Configuration

**Scenes tested:** Cornell Box, San Miguel, Rungholt
- Resolution: 1920×1080
- Sampling: 8 SPP × 10 accumulation frames = 80 total samples
- Runs: 5 iterations per configuration
- Timing: GPU-only (timestamp queries, excluding compilation and CPU overhead)

**Scenes excluded:**
- **BreakfastRoom**: Setup issues unrelated to Slang
- **Salle-de-Bain**: High run-to-run variance requiring dedicated investigation

### Shader Configurations

Each scene was tested with:
- **DXR backend**: Native HLSL vs. Slang (compiled to DXIL via DXC)
- **Vulkan backend**: Native GLSL vs. Slang (compiled to SPIR-V via glslc)

All compiler flags and optimization levels remained consistent with previous posts:
- DXC: `-O3`, Shader Model `lib_6_6`
- glslc: `-O --target-env=vulkan1.2`, SPIR-V 1.5
- Slang: v2025.12.1, `SLANG_OPTIMIZATION_LEVEL_HIGH`, pass-through to native compilers

## Results

### Performance Data by Scene

**Table: Cornell Box Performance (ms total, lower is better)**

| GPU | DXR: HLSL | DXR: Slang | Δ | Vulkan: GLSL | Vulkan: Slang | Δ |
|-----|----------:|-----------:|--:|-------------:|--------------:|--:|
| RTX 3070 | 13.99 | 8.19 | **+41%** | 5.87 | 4.84 | **+18%** |
| RTX 4090 | 3.98 | 2.27 | **+43%** | 1.51 | 1.75 | -16% |
| Intel B580 | 12.15 | 13.33 | -10% | 15.48 | 10.87 | **+30%** |
| AMD RX 7900 XT | 4.94 | 4.76 | +4% | 5.11 | 9.60 | **-88%** |

*Note: Δ represents performance gain (positive = Slang faster, negative = Slang slower)*

**Table: San Miguel Performance (ms total, lower is better)**

| GPU | DXR: HLSL | DXR: Slang | Δ | Vulkan: GLSL | Vulkan: Slang | Δ |
|-----|----------:|-----------:|--:|-------------:|--------------:|--:|
| RTX 3070 | 54.82 | 47.23 | **+14%** | 43.79 | 41.96 | +4% |
| RTX 4090 | 13.32 | 10.43 | **+22%** | 10.31 | 9.94 | +4% |
| Intel B580 | 48.79 | 47.53 | +3% | 96.32 | 78.18 | **+19%** |
| AMD RX 7900 XT | 39.79 | 39.70 | ~0% | 38.81 | 45.96 | -18% |

*Note: Δ represents performance gain (positive = Slang faster, negative = Slang slower)*

**Table: Rungholt Performance (ms total, lower is better)**

| GPU | DXR: HLSL | DXR: Slang | Δ | Vulkan: GLSL | Vulkan: Slang | Δ |
|-----|----------:|-----------:|--:|-------------:|--------------:|--:|
| RTX 3070 | 25.50 | 17.81 | **+30%** | 13.97 | 12.78 | **+9%** |
| RTX 4090 | 15.85 | 4.62 | **+71%** | 3.61 | 2.85 | **+21%** |
| Intel B580 | 22.75 | 20.83 | **+8%** | 27.49 | 22.43 | **+18%** |
| AMD RX 7900 XT | 18.99 | 18.93 | ~0% | 19.19 | 20.06 | -5% |

*Note: Δ represents performance gain (positive = Slang faster, negative = Slang slower). The RTX 4090 Rungholt DXR baseline shows unusually high variance (30.89ms, 7.03ms, 26.77ms, 7.23ms, 7.33ms across runs), suggesting measurement anomalies. The +71% improvement should be interpreted cautiously.*

## Discussion

### What the Data Shows

The performance story is **hardware and backend dependent**. Slang's optimization benefits appear most pronounced on NVIDIA's DXR path, with varying results elsewhere. Importantly, this doesn't diminish Slang's core value proposition—it clarifies it.

Across all four GPUs, spanning three vendors and multiple architectural generations, Slang delivered functionally identical rendering results. Visual parity was never in question. The ray-traced images were pixel-perfect matches to their native HLSL/GLSL counterparts. Performance varied, but remained competitive in most configurations.

### Portability vs. Performance

The variance in results highlights an important reality about cross-platform development which is,  different vendors optimize their compiler stacks differently. NVIDIA's DXC path appears to benefit significantly from Slang's frontend optimizations (as explored in [Post #2](/posts/2025-11-27-slang-performance-investigation/)). AMD's Vulkan path showed the largest regression (-37%), suggesting Slang's backend may need additional tuning for this configuration. Intel shows interesting asymmetry—their Vulkan path benefits from Slang more than their DXR path.

None of this makes Slang "better" or "worse" than native shaders in absolute terms. It means the value proposition shifts depending on context:
- Where native compilers have optimization gaps, Slang can compensate
- Where native compilers are mature, Slang matches their performance
- In a few specific cases, Slang introduces overhead that needs investigation

### The Development Trade-Off

Even in configurations where Slang matches (rather than exceeds) native performance, the single-codebase reality matters. For my ChameleonRT testbed, eliminating the dual-shader maintenance burden was valuable independent of performance considerations. The fact that I also gained 30% on my primary development hardware (RTX 3070 + DXR) reinforced the decision, but the portability argument would hold even at performance parity.

The question for any team isn't whether Slang can be faster—it's whether the specific hardware and backend combinations they target benefit from its optimizations, and whether portability outweighs potential regressions in specific configurations.

### Looking Forward

The variance in results across vendors suggests opportunities for improvement rather than fundamental limitations. For organizations investing in Slang adoption, vendor involvement in the Slang/Khronos working groups could help optimize backend code generation for specific architectures, and backend tuning for vendor-specific quirks may close performance gaps in configurations that currently show regressions. These optimizations take time and collaboration, but they're not blocked by any fundamental technical constraint. The fact that Slang achieves strong results on some vendor/backend combinations while showing regressions on others suggests the underlying approach is sound. It's a matter of tuning and optimization rather than a fundamental architectural mismatch.

## Limitations & Future Directions

### What I Didn't Test

Several important dimensions remain unexplored:

**Driver version sensitivity**: I tested one driver version per vendor. Compiler optimizations evolve with each driver release, and results could shift significantly across versions. A proper investigation would test multiple driver versions per vendor to understand sensitivity.

**Production shader complexity**: Our test scenes are representative of path tracing workloads but don't cover the full spectrum of shader patterns used in production engines. More complex material systems, procedural generation, or compute-heavy shaders might show different characteristics.

**Statistical rigor**: Professional benchmarking would require 20+ runs per configuration, controlled environments, and statistical analysis (confidence intervals, outlier detection). Our 5-run methodology provides directional data but not the rigor needed for definitive conclusions.

**Mobile and ARM architectures**: Apple M-series (Metal backend) and Qualcomm Adreno would be valuable for rounding out the portability story. These represent significant portions of the GPU market, particularly for mobile and emerging desktop platforms. Access to this hardware would provide important data points.


## Conclusions

Three posts in, the pattern is clear: **cross-vendor portability works reliably, performance varies by hardware.**

Slang delivers on its [Write Shaders Once, Run Anywhere](https://github.com/shader-slang/slang?tab=readme-ov-file#write-shaders-once-run-anywhere) promise for the backends tested. Across four GPUs from three vendors running DXR and Vulkan, visual parity was never in question. Performance varied from significant gains (NVIDIA DXR: +28% to +45%) to regressions in specific cases (AMD Vulkan: -37%), but remained competitive in most configurations.

The practical implication is that teams should test on their target hardware rather than assume universal speedups.

For my testbed, the portability alone justified the switch. Maintaining separate HLSL and GLSL implementations was a development tax I'm happy to eliminate. The performance gains made the decision easier, but the argument holds even at parity.

Slang represents a legitimate path forward for cross-platform shader development. Not a silver bullet, but a viable option that shifts the trade-off from "portability vs. performance" to "which hardware benefits most." That's meaningful progress.

## Acknowledgments

Special thanks to the anonymous friends who lent their hardware for several hours of benchmarking. Multi-vendor testing is challenging for individual developers, and this work wouldn't have been possible without their generosity and trust. Running the same code on four different GPUs from three vendors provided invaluable insights that would have been impossible otherwise.

---

## Appendix: Raw Benchmark Data

For reproducibility and further analysis, the complete raw results are provided below. Each measurement represents total render time (8 SPP × 10 accumulation frames = 80 total samples) averaged across 5 runs at 1920×1080.

### Cornell Box - Raw Timings (ms total)

| GPU | HLSL (Run 1-5) | HLSL Avg | Slang (Run 1-5) | Slang Avg | GLSL (Run 1-5) | GLSL Avg | Slang-VK (Run 1-5) | Slang-VK Avg |
|-----|----------------|----------|-----------------|-----------|----------------|----------|--------------------|---------------|
| RTX 3070 | 14.73, 13.87, 13.89, 13.60, 13.87 | 13.99 | 8.38, 8.30, 7.94, 8.40, 7.91 | 8.19 | 5.57, 6.00, 5.89, 5.90, 5.98 | 5.87 | 4.66, 4.92, 4.79, 4.91, 4.91 | 4.84 |
| RTX 4090 | 4.01, 3.98, 4.02, 4.03, 3.85 | 3.98 | 2.27, 2.27, 2.27, 2.28, 2.27 | 2.27 | 1.47, 1.47, 1.54, 1.53, 1.54 | 1.51 | 1.08, 1.08, 4.17, 1.25, 1.17 | 1.75 |
| Intel B580 | 11.12, 11.21, 8.52, 15.11, 14.81 | 12.15 | 12.87, 12.38, 13.06, 13.77, 14.56 | 13.33 | 14.67, 14.70, 13.85, 17.70, 16.48 | 15.48 | 10.82, 10.98, 10.65, 11.05, 10.87 | 10.87 |
| AMD 7900 XT | 4.46, 5.12, 5.02, 5.06, 5.01 | 4.94 | 4.74, 4.74, 4.75, 4.82, 4.75 | 4.76 | 5.24, 5.04, 5.10, 5.15, 5.01 | 5.11 | 9.54, 9.79, 9.54, 9.68, 9.46 | 9.60 |

### San Miguel - Raw Timings (ms total)

| GPU | HLSL (Run 1-5) | HLSL Avg | Slang (Run 1-5) | Slang Avg | GLSL (Run 1-5) | GLSL Avg | Slang-VK (Run 1-5) | Slang-VK Avg |
|-----|----------------|----------|-----------------|-----------|----------------|----------|--------------------|---------------|
| RTX 3070 | 54.41, 54.50, 56.33, 54.43, 54.45 | 54.82 | 47.29, 47.14, 47.27, 47.22, 47.22 | 47.23 | 43.62, 43.84, 43.95, 43.72, 43.83 | 43.79 | 41.90, 42.04, 42.13, 41.72, 42.02 | 41.96 |
| RTX 4090 | 13.44, 13.17, 13.14, 13.40, 13.44 | 13.32 | 10.31, 10.28, 10.55, 10.34, 10.66 | 10.43 | 10.47, 10.48, 9.70, 10.63, 10.24 | 10.31 | 9.82, 9.99, 10.21, 9.85, 9.83 | 9.94 |
| Intel B580 | 46.67, 50.19, 50.44, 46.64, 50.03 | 48.79 | 48.16, 46.18, 49.95, 49.33, 44.06 | 47.53 | 85.59, 93.76, 102.70, 99.55, 100.02 | 96.32 | 78.44, 78.29, 77.19, 78.56, 78.43 | 78.18 |
| AMD 7900 XT | 39.68, 39.62, 39.50, 40.03, 40.10 | 39.79 | 39.53, 39.76, 39.69, 39.93, 39.62 | 39.70 | 38.87, 38.73, 38.79, 38.85, 38.80 | 38.81 | 45.89, 45.97, 46.03, 45.87, 46.07 | 45.96 |

### Rungholt - Raw Timings (ms total)

| GPU | HLSL (Run 1-5) | HLSL Avg | Slang (Run 1-5) | Slang Avg | GLSL (Run 1-5) | GLSL Avg | Slang-VK (Run 1-5) | Slang-VK Avg |
|-----|----------------|----------|-----------------|-----------|----------------|----------|--------------------|---------------|
| RTX 3070 | 25.43, 25.78, 25.37, 25.42, 25.51 | 25.50 | 17.80, 17.78, 17.80, 17.86, 17.82 | 17.81 | 13.99, 13.97, 13.95, 13.93, 14.01 | 13.97 | 12.69, 12.80, 12.79, 12.84, 12.77 | 12.78 |
| RTX 4090 | 30.89, 7.03, 26.77, 7.23, 7.33 | 15.85 | 4.63, 4.62, 4.61, 4.63, 4.62 | 4.62 | 3.67, 3.39, 3.44, 4.10, 3.43 | 3.61 | 2.84, 2.83, 2.70, 2.71, 3.18 | 2.85 |
| Intel B580 | 24.61, 24.31, 22.31, 20.84, 21.70 | 22.75 | 20.16, 19.93, 22.80, 22.40, 18.86 | 20.83 | 35.63, 25.91, 24.85, 25.53, 25.54 | 27.49 | 27.49, 21.21, 21.08, 21.00, 21.38 | 22.43 |
| AMD 7900 XT | 19.73, 19.28, 18.21, 18.09, 19.62 | 18.99 | 19.39, 18.09, 19.35, 18.88, 18.95 | 18.93 | 19.22, 19.21, 19.40, 19.01, 19.11 | 19.19 | 19.78, 20.62, 20.30, 19.81, 19.82 | 20.06 |

