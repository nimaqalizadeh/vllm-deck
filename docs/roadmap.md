# Roadmap

The build is **re-sequenced for learning**. The original README phases bundled four hard Rust topics
(async networking, NVML/FFI, the Docker API, a full-screen TUI) into the same week. Here, each hard
concept is introduced **alone**, and there is always a runnable binary at the end of every milestone.

Guiding rule: **async is a tool, not a milestone.** It appears at M1, where a single HTTP request to
HuggingFace gives it a concrete reason to exist — not before (detecting a GPU is instant and
synchronous, so forcing async there would teach nothing).

Collaboration mode: **you drive, I navigate.** For each step the mentor explains the concept, the
shape, and the trade-offs; you write the code; the mentor reviews it ranked most- to least-important.

---

## M0 — `check` (synchronous skeleton)

**Goal:** a real, runnable binary that detects the local GPU and answers "does model X fit?" with a
weights-only estimate. No async, no TUI, no Docker.

**Deliverable:** `vllm-deck check <model>` prints GPU info + a fit verdict.

**Steps**
1. `cargo init` in place; minimal `Cargo.toml` (deps: `clap`, `anyhow`, `thiserror`, `serde`,
   `serde_json`; `nvml-wrapper` under `[target.'cfg(target_os = "linux")'.dependencies]`).
2. `src/main.rs` — entry point + subcommand routing.
3. `src/cli.rs` — clap `#[derive(Parser)]` + a `Commands` enum (just `Check` to start).
4. `src/gpu/mod.rs`, `src/gpu/detect.rs` — real NVML path gated by `#[cfg(target_os = "linux")]`;
   a stub for other OSes returning "GPU detection unavailable on this platform".
5. `src/gpu/vram.rs` — `weights_bytes = params × bytes_per_dtype`, with a **unified-memory branch**;
   output a `FitResult` struct (fits/doesn't + breakdown).

**Learning objectives:** binary vs library crate · Cargo deps, caret versions & feature flags ·
file-based modules (`mod` vs `pub mod`) · clap derive · `struct`/`enum` modeling · `Result<T, E>`,
the `?` operator, `anyhow` vs `thiserror` · conditional compilation (`#[cfg]` attribute vs `cfg!()`
macro) · first taste of ownership/borrowing.

**Reuse from references:** `nvml_optional()` in
`github/spark-dashboard/src/metrics/gpu.rs` (treat `NotSupported`/`InvalidArg` as "absent", not a
crash); `GpuInfo`/`SystemSpecs` shape in `github/llmfit/llmfit-core/src/hardware.rs`.

**Verify**
- macOS: `cargo build` + `cargo run -- check <model>` succeed via the stub path.
- ASUS RTX 4060 (Linux): reports a discrete GPU with separate ~8 GB VRAM → *discrete branch*.
- DGX Spark: reports GB10 + unified memory pool → *unified branch*. Compare the two outputs.

---

## M1 — `search` (async enters)

**Goal:** query the HuggingFace model API and print results as a headless table.

**Deliverable:** `vllm-deck search "llama 3"` lists matching models (name, downloads, likes).

**New concepts:** the `tokio` async runtime · `async fn` / `.await` · `reqwest` HTTP client · one GET
request, then `serde` deserialization of JSON into typed structs · `#[tokio::main]`.

**Reuse:** `github/rust-hf-downloader/src/api.rs`, `headless.rs`.

**Verify:** real results returned on any networked machine; handles "no results" and HTTP errors.

---

## M2 — `download` (async concurrency)

**Goal:** download model weights with progress, resume, and verification.

**Deliverable:** `vllm-deck download <model>` fetches files into a local cache.

**New concepts:** concurrent in-flight downloads (`futures`/`tokio` tasks) · streaming response bodies ·
progress reporting · resumable downloads (HTTP range requests) · checksum verification (`sha2`).

**Reuse:** `github/rust-hf-downloader/src/download.rs`.

**Verify:** a small model downloads; interrupting and re-running resumes rather than restarting.

---

## M3 — `deploy` (Docker Engine API)

**Goal:** pull and launch a vLLM container, wait until healthy.

**Deliverable:** `vllm-deck deploy <model>` runs vLLM with GPU passthrough on a local model.

**New concepts:** `bollard` (Docker Engine API client) · container create/start/stop/logs · GPU
device passthrough · volume mounts + port mapping · readiness/health polling · `toml` config profiles
(`~/.config/vllm-deck/`).

**Open question resolved here:** the DGX Spark is aarch64, so the stock `vllm/vllm-openai` x86 image
won't run. Identify the correct ARM64 image (likely an NVIDIA NGC vLLM container) and pin it per-arch.

**Verify:** container reaches healthy state on the ASUS 4060; repeat on the Spark with the ARM64 image.

---

## M4 — `monitor` / `dashboard` (TUI)

**Goal:** a real-time terminal dashboard of vLLM + GPU metrics.

**Deliverable:** `vllm-deck monitor` shows live panels; `dashboard` combines all views.

**New concepts:** `ratatui` widgets + layout · `crossterm` raw mode / event handling · an async event
loop (input + a metrics-poll tick) · scraping vLLM's Prometheus `/metrics` (`prometheus-parse`) · live
NVML stats · sparklines for history.

**Reuse:** `github/spark-dashboard/src/engines/vllm.rs`, `metrics/gpu.rs`.

**Verify:** dashboard updates live against a running vLLM container on both Linux machines.

---

## After M4 (polish)

- Refactor shared logic into a `core` library crate (workspace) once CLI + TUI both consume it —
  see [ADR-0001](adr/0001-single-crate-start.md).
- Better error messages, config profiles, demo GIF, publish to crates.io.
