# Project Structure

## Principle: grow the structure, don't pre-build it

vllm-deck **starts as a single binary crate** and grows toward the target layout one milestone at a
time. We do not scaffold empty modules for features we haven't reached. Rationale and trade-offs:
[ADR-0001](adr/0001-single-crate-start.md).

## Current structure (M0)

Only what `check` needs exists:

```
vllm-deck/
├── Cargo.toml           # manifest; deps added per milestone
├── src/
│   ├── main.rs          # entry point + subcommand routing
│   ├── cli.rs           # clap Parser + Commands enum (just `Check` for now)
│   └── gpu/
│       ├── mod.rs       # re-exports; declares detect + vram
│       ├── detect.rs    # NVML detection (cfg-gated Linux) + non-Linux stub
│       └── vram.rs      # weights-only estimate + FitResult; unified-memory branch
├── README.md
├── LICENSE
├── docs/                # this documentation
└── github/              # reference projects (read-only, for inspiration)
```

## Target structure (end state, reached gradually)

```
src/
├── main.rs              # entry point, subcommand routing
├── cli.rs               # clap subcommand definitions
├── config.rs            # TOML config load/save (~/.config/vllm-deck/)   [M3]
├── models.rs            # shared data types
├── hf/                  # HuggingFace integration                        [M1–M2]
│   ├── api.rs           # search, metadata, download URLs
│   ├── download.rs      # chunked parallel downloader
│   └── cache.rs         # local model cache
├── gpu/                 # GPU hardware layer (cfg-gated NVML)             [M0]
│   ├── detect.rs
│   └── vram.rs
├── docker/              # container orchestration                        [M3]
│   ├── vllm.rs
│   ├── webui.rs
│   └── health.rs
├── monitor/             # metrics & monitoring                           [M4]
│   ├── metrics.rs
│   └── gpu_stats.rs
└── ui/                  # TUI layer                                       [M4]
    ├── app.rs
    ├── render.rs
    ├── views/           # search / check / deploy / monitor
    └── events.rs
```

## Module conventions

- **File-based modules.** A folder `gpu/` with `gpu/mod.rs` is the module `gpu`; files inside it
  (`detect.rs`, `vram.rs`) are submodules declared in `mod.rs` via `mod detect;` / `mod vram;`.
- **`mod` vs `pub mod`.** `mod x;` includes the module but keeps it private to its parent; `pub mod x;`
  also re-exports it so callers outside the parent can reach it. In a binary (`main.rs`) most modules
  stay plain `mod` — nothing external links to a binary. When we extract a `core` *library* later,
  the items that library exposes become `pub`.
- **The `core` extraction (later).** Once both the CLI and the TUI consume the same logic (GPU
  detection, memory math, HF client), that logic moves into a `vllm-deck-core` library crate and the
  binary becomes a thin shell over it — mirroring `github/llmfit`'s `llmfit-core` + frontends layout.
  Timing: when duplication or the need to share actually appears, not before.
