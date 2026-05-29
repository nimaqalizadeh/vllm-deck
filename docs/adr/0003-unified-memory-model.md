# ADR-0003: Model unified memory explicitly, not just discrete VRAM

- **Status:** accepted
- **Date:** 2026-05-29

## Context

The primary target, the NVIDIA DGX Spark (GB10), uses **unified memory**: CPU and GPU share one
~128 GB LPDDR5X pool. There is no separate "VRAM" number to fit a model into. A discrete GPU like the
RTX 4060, by contrast, has its own ~8 GB of VRAM separate from system RAM. The README's framing —
"does it fit my VRAM?" — silently assumes the discrete case.

Both reference repos already encode this distinction: `unified_memory: bool` and `total_gpu_vram_gb`
in `github/llmfit/llmfit-core/src/hardware.rs`, and spark-dashboard's README notes "a single unified
pool ... e.g. DGX Spark GB10, GH200".

## Decision

The memory model carries an explicit `unified_memory: bool`, and the fitness estimator branches on it:

- **Discrete:** fit = weights + KV cache + activations ≤ free VRAM (the GPU's own pool).
- **Unified:** the model competes with the OS and all other processes for the single shared pool;
  the budget and the "available" figure come from the unified pool, not a dedicated VRAM number.

## Rationale & trade-offs

- **For:** correctness on the actual target hardware; this is a genuine differentiator (`llmfit` is
  aware of it, but most vLLM-specific tools are not).
- **Against:** more branching and two mental models to keep straight. Accepted — it's inherent to the
  hardware, not incidental complexity.

## Consequences

`gpu::detect` must report `unified_memory` (NVML can indicate this), and `gpu::vram` must never assume
a separate VRAM pool. Both Linux test machines exist specifically to exercise both branches.

**NVML caveat — validate before trusting the number.** On a unified GB10, `nvmlDeviceGetMemoryInfo`'s
`total`/`free` report the GPU's *view* of the shared LPDDR5X pool; that is **not** the same as the
budget actually allocatable to a model, because the OS and every other process contend for the same
pool. Treating NVML's `total` as "VRAM you can fill" is exactly the unified-memory bug this ADR exists
to avoid. Therefore:

- In **M0**, capture the *raw* NVML output on the real Spark (the M0 Verify step compares Spark vs 4060
  — make examining the actual `total`/`free`/`used` values the explicit goal, not just "it ran").
- On the unified branch, cross-check the NVML figure against system memory rather than assuming a
  dedicated VRAM number, and derive the available budget from the shared pool minus what is already in
  use — do not hardcode or trust a single "total memory" field.
