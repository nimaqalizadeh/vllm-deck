# ADR-0004: Gate NVML behind `#[cfg(target_os = "linux")]` with a stub elsewhere

- **Status:** accepted
- **Date:** 2026-05-29

## Context

NVML (NVIDIA Management Library) is a C library that ships with the NVIDIA driver and exists only on
Linux and Windows. The `nvml-wrapper` crate provides safe Rust bindings to it. The owner edits on
macOS (no NVIDIA GPU, no NVML) and tests on two Linux machines. If `nvml-wrapper` were an
unconditional dependency, the project would fail to build on the Mac.

## Decision

1. Declare `nvml-wrapper` under `[target.'cfg(target_os = "linux")'.dependencies]` so it is only
   compiled on Linux.
2. In `gpu/detect.rs`, gate the real NVML code with `#[cfg(target_os = "linux")]` and provide a
   `#[cfg(not(target_os = "linux"))]` **stub** with the same function signatures, returning a
   "GPU detection unavailable on this platform" result.
3. The rest of the codebase depends only on OS-neutral types (`GpuInfo`, `FitResult`), never on
   `nvml-wrapper` directly.

Pattern adapted from `github/spark-dashboard/src/metrics/gpu.rs`.

## Rationale & trade-offs

- **For:** the project builds and runs everywhere (Mac included); the GPU library is isolated to one
  module; a future ROCm/Metal backend can slot in behind the same neutral interface.
- **Against:** two code paths to keep in sync (real + stub). Mitigated by keeping the stub tiny and
  the neutral interface small.
- **Note:** NVML also works on Windows. We gate on `linux` only because that is the stated target;
  widening to `any(target_os = "linux", target_os = "windows")` is a one-line change if needed.

## Consequences

NVML errors that mean "this query isn't supported on this GPU" (`NotSupported`, `InvalidArg`) are
mapped to `None` rather than propagated as failures, so partial GPU support degrades gracefully.
