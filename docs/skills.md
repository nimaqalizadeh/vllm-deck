# Skills Map

This project doubles as a structured Rust curriculum. Each milestone introduces a focused set of
concepts; nothing is used before it's been introduced. This file is also the "resume map" — the
concrete, demonstrable skills the finished tool evidences.

## Rust concepts by milestone

| Milestone | Rust concepts | Ecosystem / crates |
|---|---|---|
| **M0 `check`** | crates (binary vs library), modules (`mod`/`pub mod`), `struct`/`enum`, pattern matching, `Result`/`Option`, `?` operator, ownership & borrowing basics, conditional compilation (`#[cfg]` vs `cfg!()`) | `clap` (derive), `anyhow`, `thiserror`, `serde`/`serde_json`, `nvml-wrapper` |
| **M1 `search`** | `async`/`.await`, the runtime concept, `Future`, deserialization into typed structs, lifetimes in passing | `tokio`, `reqwest`, `serde` |
| **M2 `download`** | concurrency (spawning tasks, `join`), streams, shared state (`Arc`), backpressure, traits in practice | `tokio`, `futures`, `reqwest` (stream), `sha2` |
| **M3 `deploy`** | builder patterns, larger API surfaces, config (de)serialization, polling/retry, the type-state idea | `bollard`, `toml`, `serde` |
| **M4 `monitor`** | event loops, the draw/update cycle, channels, `Drain`/terminal teardown (RAII), trait objects for widgets | `ratatui`, `crossterm`, `prometheus-parse` |

## Cross-cutting skills (developed throughout)

- **Error handling strategy** — where to define typed errors (`thiserror`) vs propagate with context
  (`anyhow`), and why mixing them at the right layer matters.
- **FFI / platform isolation** — wrapping a C library (NVML) behind a safe, OS-neutral interface and
  keeping it from leaking into the rest of the code.
- **Cross-platform builds** — `cargo`/Cargo.toml target-specific dependencies; degrading gracefully
  when hardware is absent.
- **API design** — small neutral types (`GpuInfo`, `FitResult`) as seams between layers.
- **Reading real codebases** — adapting patterns from `github/` references rather than copying them.

## Resume framing (what this demonstrates)

- "Built a cross-platform Rust CLI/TUI (single static binary) for the vLLM deployment lifecycle."
- "Integrated a C library (NVML) via safe Rust bindings, gated by conditional compilation so the tool
  builds and runs on machines without an NVIDIA GPU."
- "Modeled both discrete-VRAM and unified-memory GPUs (NVIDIA DGX Spark GB10), correctly estimating
  model fit where most tools assume a dedicated VRAM pool."
- "Used async Rust (`tokio`) for HTTP, concurrent downloads, the Docker Engine API, and a live metrics
  dashboard."
- Tested on real hardware: NVIDIA DGX Spark (aarch64, GB10) and an RTX 4060 (x86_64, discrete).

## Note on the `skills/` directory in `github/llmfit`

The reference repo `llmfit` ships a `skills/` folder — those are *Claude/agent skills* (tool
instructions), unrelated to this learning skills map. Don't confuse the two.
