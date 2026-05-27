# Architecture

## Layers

vllm-deck is organized as thin, mostly-independent layers behind a single binary. Each maps to a
milestone and can be exercised on its own.

```
              ┌─────────────────────────────────────────────┐
   user  ───▶ │  cli.rs (clap)  →  subcommand dispatch       │
              └───────────────┬─────────────────────────────┘
                              │
        ┌─────────────┬───────┴───────┬──────────────┬──────────────┐
        ▼             ▼               ▼              ▼              ▼
     gpu/          hf/             docker/        monitor/         ui/
   detect+vram   api+download    vllm+health   metrics+stats   ratatui TUI
   (NVML, sync)  (reqwest, async)(bollard,async)(reqwest,async) (crossterm)
        │             │               │              │
        └─────────────┴───────┬───────┴──────────────┘
                              ▼
                      external systems
              NVML · HuggingFace API · Docker daemon · vLLM /metrics
```

## Data flow by command

- **`check`** → `gpu::detect` reads the local GPU (NVML) → `gpu::vram` combines that with the model's
  `config.json` (params, dtype) → `FitResult`. Fully synchronous.
- **`search`** → `hf::api` issues an async HTTP GET → `serde` parses JSON → table output.
- **`download`** → `hf::api` resolves file list → `hf::download` streams files concurrently → cache.
- **`deploy`** → (optionally `download`) → `gpu::vram` pre-check → `docker::vllm` creates/starts the
  container → `docker::health` polls until ready.
- **`monitor`** → `monitor::metrics` scrapes vLLM `/metrics` + `monitor::gpu_stats` reads NVML on a
  tick → `ui` renders. `dashboard` wires all views together.

## The GPU boundary (cfg-gated)

The GPU layer is the one place that touches a platform C library, so it is isolated behind a small,
OS-neutral interface. NVML calls live under `#[cfg(target_os = "linux")]`; a stub provides the same
function signatures elsewhere, returning "detection unavailable" instead of failing to compile.

Why this matters: the rest of the code (`vram.rs`, `cli.rs`, the future TUI) depends only on the
neutral types (`GpuInfo`, `FitResult`), never on `nvml-wrapper` directly. So the macOS build works,
and a future AMD/ROCm or Apple-Metal backend could slot in behind the same interface. Detail:
[ADR-0004](adr/0004-cfg-gated-nvml.md).

NVML calls are also wrapped so that `NotSupported`/`InvalidArg` become `None` rather than errors —
some queries are simply N/A on a given GPU (pattern borrowed from
`github/spark-dashboard/src/metrics/gpu.rs`).

## Memory model: unified vs discrete

The fitness check branches on a single fact — `unified_memory: bool`:

- **Discrete (RTX 4060):** the GPU has its own VRAM (~8 GB) separate from system RAM. "Fit" means
  weights + KV cache + activations ≤ free VRAM.
- **Unified (DGX Spark GB10):** CPU and GPU share one ~128 GB LPDDR5X pool. There is no separate VRAM
  number; the model competes with the OS and everything else for the single budget.

Treating these the same is the bug most tools have on the Spark. Detail:
[ADR-0003](adr/0003-unified-memory-model.md).

## Async boundary

Synchronous: `gpu` (instant NVML reads), the memory math, CLI parsing.
Asynchronous: everything doing I/O that waits — `hf` (HTTP), `docker` (Engine API), `monitor`
(scrape loop), and the TUI event loop. `tokio` is introduced at M1 and the binary's `main` becomes
`#[tokio::main]` from that point. Rationale for the timing: [ADR-0002](adr/0002-resequenced-milestones.md).

## Error handling

- Library-style modules define precise error types with `thiserror`.
- The application edge (`main.rs`, command handlers) uses `anyhow` to propagate "any error" with
  human-readable context (e.g. `.context("reading model config.json")`).
