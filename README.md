# vllm-deck

A unified Rust **CLI + terminal dashboard (TUI)** for searching, fitness-checking, deploying, and
monitoring [vLLM](https://github.com/vllm-project/vllm) inference servers on Linux machines with
NVIDIA GPUs — with the **NVIDIA DGX Spark (GB10)** as the primary target.

> **Status: work in progress / learning project.** This is being built milestone by milestone (see
> [Roadmap](#roadmap)). It is also a deliberate vehicle for learning idiomatic Rust — CLI, async, FFI
> to a C library, and TUI. The detailed learning plan lives in [`docs/`](docs/).

## Problem

Today, deploying a model on vLLM requires juggling multiple tools:

1. Search for a model on HuggingFace (browser or `huggingface-cli`)
2. Manually check if the model fits your GPU memory (mental math or a web calculator)
3. Download the model weights (`huggingface-cli` or `git-lfs`)
4. Write a Docker run command or compose file for vLLM with the right flags
5. Open Grafana or `curl /metrics` to see if it's working

No single tool handles this pipeline — especially not in Rust, and especially not aware of
**unified-memory** machines like the DGX Spark.

## Solution

One binary. One workflow:

```
vllm-deck search "llama 3"                  → browse models from HuggingFace
vllm-deck check meta-llama/Llama-3-8B       → does it fit my GPU's memory?
vllm-deck deploy meta-llama/Llama-3-8B      → download + launch vLLM container
vllm-deck monitor                           → real-time TUI dashboard of vLLM metrics
vllm-deck dashboard                         → full TUI combining all of the above
```

Every non-TUI subcommand (`check`, `search`, `deploy`) also supports `--json` / `--output json` for
scripting and composability.

## Target hardware

Linux + NVIDIA is the target. Two real machines drive development and testing, deliberately chosen to
cover the two memory architectures the tool must understand:

| Machine | Arch | GPU memory model | Role |
|---|---|---|---|
| **NVIDIA DGX Spark (GB10)** | aarch64 | **Unified** ~128 GB LPDDR5X shared CPU+GPU | Primary target; validates the unified-memory path |
| **ASUS laptop, RTX 4060** | x86_64 | **Discrete** ~8 GB GDDR6, separate from system RAM | Fast dev loop; validates the discrete-VRAM path |
| macOS (Apple Silicon) | aarch64 | no NVIDIA GPU | Portability check only — builds via a stub, no live GPU data |

NVML (NVIDIA Management Library — the C library that reports GPU stats) exists only on Linux/Windows,
so the GPU layer is gated with `#[cfg(target_os = "linux")]` and stubbed elsewhere. This keeps the
project compiling on a Mac with no GPU — see [ADR-0004](docs/adr/0004-cfg-gated-nvml.md).

## Features

### 1. Model Discovery (inspired by [rust-hf-downloader](https://github.com/JohannesBertens/rust-hf-downloader))

- Search HuggingFace models by name, task, or tag
- Browse results in an interactive TUI with sorting (downloads, likes, recent)
- View model metadata: parameter count, quantization options, license, file sizes
- Download model weights with chunked parallel downloads and progress bars
- Resume interrupted downloads
- Support for gated models via HuggingFace token

**Crates:** `reqwest`, `tokio`, `serde`, `ratatui` (TUI later)

### 2. Memory Fitness Check (inspired by [llmfit](https://github.com/AlexsJones/llmfit))

- Auto-detect local GPU hardware via NVML — GPU model, total/available memory, **unified vs discrete**
- Estimate model memory requirements accounting for:
  - Model weights (parameters × bytes per dtype)
  - KV cache budget (vLLM-specific: PagedAttention block size × max sequences)
  - Activation memory during inference
  - Tensor-parallelism splits (multi-GPU)
- **Unified-memory aware:** on the DGX Spark there is no separate VRAM pool — weights, KV cache, and
  the OS share one budget. The estimator branches on `unified_memory` instead of assuming a dedicated
  VRAM number. See [ADR-0003](docs/adr/0003-unified-memory-model.md).
- Output: "fits" / "doesn't fit" with a breakdown of where memory goes
- Suggest optimal quantization / tensor-parallel config when the full model doesn't fit

**Crates:** `nvml-wrapper`, `serde_json` (for HF model config parsing)

### 3. Container Orchestration (inspired by [vllm-cli](https://github.com/Chen-zexi/vllm-cli))

- Pull and launch vLLM Docker containers via the Docker Engine API (no shell commands)
- Auto-configure: GPU passthrough, model volume mount, port mapping, tensor parallelism
- Optionally launch Open WebUI container alongside vLLM
- Manage running instances: list, stop, restart, logs
- Config profiles: save and reuse deployment configs (model, GPU, quantization, max-seq-len)
- Health check: wait for vLLM to be ready before returning
- **`--dry-run`:** print the exact image, container config, mounts, ports, and GPU flags that would be
  used — without pulling images or starting a container
- **aarch64 caveat:** the DGX Spark needs an ARM64 vLLM image (likely an NVIDIA NGC container, not the
  stock `vllm/vllm-openai`). Resolved in a pre-M3 spike — see [`docs/roadmap.md`](docs/roadmap.md).

**Crates:** `bollard` (Docker API client), `tokio`, `serde`, `toml` (config files)

### 4. Real-time Monitoring TUI (inspired by [vllm-cli](https://github.com/Chen-zexi/vllm-cli) + [spark-dashboard](https://github.com/niklasfrick/spark-dashboard))

- Scrape vLLM's Prometheus `/metrics` endpoint at configurable intervals
- Display in a ratatui dashboard:
  - Requests: running / waiting / completed / failed
  - Throughput: prompt tokens/s, generation tokens/s
  - Latency: time-to-first-token (TTFT), time-per-output-token (TPOT), e2e latency
  - KV cache utilization %
  - GPU utilization and memory usage (via NVML)
- Historical sparklines / mini-charts for key metrics
- Container logs panel (streamed from Docker)

**Crates:** `ratatui`, `crossterm`, `reqwest`, `prometheus-parse`, `nvml-wrapper`

## Roadmap

The README's original "Phase 1–5" plan front-loaded four hard things at once (async, NVML/FFI, Docker,
TUI). The build is **re-sequenced for learning** so each hard concept arrives alone and there is always
a working binary. Async is introduced at M1, where an HTTP request gives it a concrete reason to exist.

| Milestone | Command | New concept introduced | Async? |
|---|---|---|---|
| **M0** | `vllm-deck check <model>` | project skeleton, clap, `Result`/errors, modules, **cfg/FFI split** | no |
| **M1** | `vllm-deck search "<q>"` | **async + HTTP**: tokio, reqwest, `.await`, JSON deserialization | yes (single request) |
| **M2** | `vllm-deck download <model>` | **async concurrency**: streaming, multiple in-flight downloads, progress | yes |
| **M3** | `vllm-deck deploy <model>` | Docker Engine API via bollard; ARM64 image resolution | yes |
| **M4** | `vllm-deck monitor` / `dashboard` | ratatui TUI + async event loop, Prometheus scrape | yes |

Full milestone breakdowns, learning objectives, and verification steps: [`docs/roadmap.md`](docs/roadmap.md).
Decisions and their rationale: [`docs/adr/`](docs/adr/).

## Architecture

This is the **target** end-state layout. The project starts as a single binary crate and grows into
this shape (the `core` library extraction happens later — see
[`docs/project_structure.md`](docs/project_structure.md) and [ADR-0001](docs/adr/0001-single-crate-start.md)).

```
src/
├── main.rs              # Entry point, subcommand routing
├── cli.rs               # Clap subcommand definitions
├── config.rs            # TOML config load/save (~/.config/vllm-deck/)
├── models.rs            # Shared data types
│
├── hf/                  # HuggingFace integration
│   ├── api.rs           # HF Hub API client (search, metadata, download URLs)
│   ├── download.rs      # Chunked parallel downloader with progress
│   └── cache.rs         # Local model cache management
│
├── gpu/                 # GPU hardware layer (NVML, cfg-gated to Linux)
│   ├── detect.rs        # NVML-based GPU detection (+ non-Linux stub)
│   └── vram.rs          # Memory estimation engine (weights + KV cache + activations)
│
├── docker/              # Container orchestration
│   ├── vllm.rs          # vLLM container lifecycle (create, start, stop, logs)
│   ├── webui.rs         # Open WebUI container management
│   └── health.rs        # Readiness polling
│
├── monitor/             # Metrics & monitoring
│   ├── metrics.rs       # Prometheus /metrics scraper and parser
│   └── gpu_stats.rs     # Real-time GPU stats via NVML
│
└── ui/                  # TUI layer
    ├── app.rs           # Main event loop
    ├── render.rs        # Layout and widget rendering
    ├── views/           # search / check / deploy / monitor views
    └── events.rs        # Keyboard/mouse input handling
```

## Dependencies

Dependencies are **added per milestone**, not all at once — the manifest stays as small as the current
milestone needs. The full target set:

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }        # CLI
ratatui = "0.29"                                        # TUI
crossterm = { version = "0.28", features = ["event-stream"] }
tokio = { version = "1", features = ["full"] }          # async runtime
futures = "0.3"
reqwest = { version = "0.12", features = ["rustls-tls", "json", "stream"] }
serde = { version = "1", features = ["derive"] }        # serialization
serde_json = "1"
toml = "0.8"
bollard = "0.18"                                        # Docker API
prometheus-parse = "0.2"                                # monitoring
thiserror = "2"                                         # define error types
anyhow = "1"                                            # propagate errors w/ context
tracing = "0.1"
tracing-subscriber = "0.3"
sha2 = "0.10"                                           # download verification
hex = "0.4"

# GPU — Linux only, so NVML never breaks the macOS build
[target.'cfg(target_os = "linux")'.dependencies]
nvml-wrapper = "0.10"
```

## Differentiators

| vs. | GitHub | vllm-deck advantage |
|---|---|---|
| **vllm-cli** (Python) | https://github.com/Chen-zexi/vllm-cli | Single static binary, no Python/pip. Faster startup. Richer TUI. |
| **rust-hf-downloader** | https://github.com/JohannesBertens/rust-hf-downloader | Goes beyond downloading — checks memory fit, deploys, monitors. |
| **llmfit** | https://github.com/AlexsJones/llmfit | vLLM-aware memory math (KV cache, tensor parallelism), not just "do weights fit." |
| **llmserve** | https://github.com/AlexsJones/llmserve | Dedicated vLLM focus with memory pre-check and model provisioning. |
| **spark-dashboard** | https://github.com/niklasfrick/spark-dashboard | TUI-based, zero setup. No separate web frontend needed. |
| **Grafana dashboards** | https://github.com/vllm-project/vllm/tree/main/examples/online_serving/dashboards/grafana | Zero setup. No Prometheus/Grafana stack needed. Just run `vllm-deck monitor`. |

## Existing Crate Ecosystem

Everything needed already exists as a Rust crate — no need to write low-level bindings:

| Need | Crate | GitHub | Maturity |
|---|---|---|---|
| HuggingFace API | `reqwest` + `serde` (HTTP/JSON) | https://github.com/seanmonstar/reqwest | Industry standard |
| Docker API | `bollard` | https://github.com/fussybeaver/bollard | Mature, 1000+ stars |
| GPU info | `nvml-wrapper` | https://github.com/Cldfire/nvml-wrapper | Stable NVML bindings |
| TUI framework | `ratatui` | https://github.com/ratatui/ratatui | De facto standard, very active |
| Prometheus parsing | `prometheus-parse` | https://github.com/ccakes/prometheus-parse-rs | Lightweight, sufficient |
| HTTP | `reqwest` | https://github.com/seanmonstar/reqwest | Industry standard |
| CLI | `clap` | https://github.com/clap-rs/clap | Industry standard |

## Related Projects

Other projects in the vLLM ecosystem for reference:

| Project | GitHub | What it does |
|---|---|---|
| **vLLM** | https://github.com/vllm-project/vllm | The inference engine itself |
| **vLLM Router** (official, Rust) | https://github.com/vllm-project/router | Rust load balancer for vLLM fleets |
| **vLLM Production Stack** | https://github.com/vllm-project/production-stack | Official Kubernetes Helm chart |
| **vllm-studio** | https://github.com/sybil-solutions/vllm-studio | Electron control panel for vLLM |
| **vllm-playground** | https://github.com/micytao/vllm-playground | Web UI for managing vLLM servers |
| **vllm-manager** | https://github.com/amirrouh/vllm-manager | Node.js TUI for vLLM |
| **rvLLM** | https://github.com/m0at/rvllm | Full Rust rewrite of vLLM engine |
| **nviwatch** | https://github.com/msminhas93/nviwatch | Rust TUI for NVIDIA GPU monitoring |

## Documentation

| Doc | Purpose |
|---|---|
| [`instructor.md`](instructor.md) | How the AI mentor operates — role, do/don't, teaching loop, per-milestone reading discipline |
| [`docs/roadmap.md`](docs/roadmap.md) | Milestone-by-milestone build path with learning objectives & verification |
| [`docs/project_structure.md`](docs/project_structure.md) | Current vs target module layout, and the single-crate→workspace plan |
| [`docs/architecture.md`](docs/architecture.md) | Layers, data flow, the cfg-gated GPU boundary, memory model |
| [`docs/skills.md`](docs/skills.md) | Rust skills demonstrated per milestone (the learning/resume map) |
| [`docs/adr/`](docs/adr/) | Architecture Decision Records — the *why* behind each locked choice |
