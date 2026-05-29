# Roadmap

The build is **re-sequenced for learning**. The original README phases bundled four hard Rust topics
(async networking, NVML/FFI, the Docker API, a full-screen TUI) into the same week. Here, each hard
concept is introduced **alone**, and there is always a runnable binary at the end of every milestone.

Guiding rule: **async is a tool, not a milestone.** It appears at M1, where a single HTTP request to
HuggingFace gives it a concrete reason to exist — not before (detecting a GPU is instant and
synchronous, so forcing async there would teach nothing).

Collaboration mode: **you drive, I navigate.** For each step the mentor explains the concept, the
shape, and the trade-offs; you write the code; the mentor reviews it ranked most- to least-important.

**How to read the steps:** every milestone is split into the smallest shippable units (`M0-1`,
`M0-2`, …). Each step lists *what to do* and *done when*. Do them in order; don't start the next
step until the current one's "done when" is true.

**Tracking progress:** each step is a checkbox. Tick it (`- [x]`) once its "done when" holds. The
first unchecked box is the current step — this is how a fresh session (or a new AI context) picks up
exactly where the build left off, so keep the boxes honest.

---

## M-1 — Project Infrastructure & CI Baseline

**Goal:** Establish rigorous standards for code quality and testing from Day 1 to prevent technical debt.

**New concepts:** CI/CD best practices vs local task runners (`Justfile`) · Git pre-commit hooks · enforcing 100 max width and `2021` edition via `rustfmt`.

**Steps**

- [ ] **M-1-1 — `rust-toolchain.toml`.** Pin the toolchain channel (stable) and the `2021` edition baseline so every machine builds with the same compiler. *Done when:* `rustup show` reports the pinned version inside the repo.
- [ ] **M-1-2 — `rustfmt.toml`.** Set `max_width = 100` and edition `2021`. *Done when:* `cargo fmt --check` runs and reports no diff on an empty/placeholder source tree.
- [ ] **M-1-3 — `clippy.toml` + lint policy.** Decide the lint level (e.g. `clippy::all` + `-D warnings`). *Done when:* `cargo clippy -- -D warnings` passes.
- [ ] **M-1-4 — `Justfile`.** Add recipes: `just fmt`, `just lint`, `just test`, `just check` (runs all three). *Done when:* each recipe runs locally.
- [ ] **M-1-5 — `.pre-commit-config.yaml`.** Wire pre-commit to run fmt + clippy before each commit. *Done when:* `pre-commit run --all-files` passes and a bad-format commit is blocked.
- [ ] **M-1-6 — `.github/workflows/ci.yml`.** Baseline workflow: checkout, install toolchain, run fmt-check, clippy, test. *Done when:* pushing to `main` triggers a green CI run.

**Verify:** Committing to `main` triggers a successful GitHub Actions CI run that checks formatting, clippy warnings, and runs tests.

---

## M0 — `check` (synchronous skeleton)

**Goal:** a real, runnable binary that detects the local GPU and answers "does model X fit?" with a
weights-only estimate. No async, no TUI, no Docker.

**Deliverable:** `vllm-deck check <model>` prints GPU info + a fit verdict.

**New concepts:** binary vs library crate · Cargo deps, caret versions & feature flags · file-based modules (`mod` vs `pub mod`) · clap derive · `struct`/`enum` modeling · `Result<T, E>`, `?`, `anyhow` vs `thiserror` · conditional compilation (`#[cfg]` vs `cfg!()`) · ownership/borrowing · unit tests (`#[test]`).

**Steps**

- [ ] **M0-1 — `cargo init` + `Cargo.toml`.** Init in place; add deps `clap`, `anyhow`, `thiserror`, `serde`, `serde_json`; put `nvml-wrapper` under `[target.'cfg(target_os = "linux")'.dependencies]`. *Done when:* `cargo build` succeeds.
- [ ] **M0-2 — `src/main.rs`.** Entry point that parses args and routes to a subcommand handler. *Done when:* `cargo run -- --help` prints usage.
- [ ] **M0-3 — `src/cli.rs`.** clap `#[derive(Parser)]` with a `Commands` enum containing just `Check { model: String }`. *Done when:* `cargo run -- check llama` parses and reaches a handler stub.
- [ ] **M0-4 — `src/gpu/mod.rs` + `src/gpu/detect.rs`.** Real NVML detection gated by `#[cfg(target_os = "linux")]`; a stub for other OSes returning "GPU detection unavailable on this platform". Reuse `nvml_optional()` from `github/spark-dashboard/src/metrics/gpu.rs` (treat `NotSupported`/`InvalidArg` as "absent", not a crash). *Done when:* `check` prints GPU info on Linux and the stub message on macOS.
- [ ] **M0-5 — `GpuInfo` model.** Define the struct (name, total VRAM, unified vs discrete). Borrow the shape from `github/llmfit/llmfit-core/src/hardware.rs`. *Done when:* detection populates `GpuInfo`.
- [ ] **M0-6 — `src/gpu/vram.rs` fit math.** `weights_bytes = params × bytes_per_dtype`, with a **unified-memory branch**; return a `FitResult` (fits/doesn't + breakdown). *Done when:* `check` prints a fit verdict.
- [ ] **M0-7 — Unit tests for the memory math.** Cover discrete, unified, and the doesn't-fit case. *Done when:* `cargo test` passes.
- [ ] **M0-8 — Global `--json` / `--output json` flag.** Add a top-level output flag so subcommands can emit machine-readable output for scripting; `check --json` serializes the `FitResult` (GPU info + verdict + breakdown) as JSON, while the default stays the human-readable table. *Done when:* `check --json <model>` prints valid, parseable JSON.

**Verify**
- macOS: `cargo build` + `cargo run -- check <model>` succeed via the stub path.
- ASUS RTX 4060 (Linux): reports a discrete GPU with separate ~8 GB VRAM → *discrete branch*.
- DGX Spark: reports GB10 + unified memory pool → *unified branch*. Compare the two outputs.
- Unit tests for the memory math logic pass.

---

## M1 — `search` (async enters)

**Goal:** query the HuggingFace model API and print results as a headless table.

**Deliverable:** `vllm-deck search "llama 3"` lists matching models (name, downloads, likes).

**New concepts:** the `tokio` async runtime · `async fn` / `.await` · `reqwest` HTTP client · one GET request, then `serde` deserialization · `#[tokio::main]` · structured logging (`tracing`) · offline graceful degradation.

**Steps**

- [ ] **M1-1 — Add async deps.** `tokio` (with needed features), `reqwest`, `tracing`, `tracing-subscriber`. *Done when:* `cargo build` succeeds.
- [ ] **M1-2 — `#[tokio::main]` + async routing.** Convert `main` to async and make the subcommand handler async. *Done when:* the existing `check` command still works under the async runtime.
- [ ] **M1-3 — `Search` subcommand.** Add `Search { query: String }` to the `Commands` enum. *Done when:* `vllm-deck search "llama 3"` parses and reaches a handler.
- [ ] **M1-4 — HF API call.** One GET to the HuggingFace model search endpoint via `reqwest`. Reuse `github/rust-hf-downloader/src/api.rs`. *Done when:* the raw response comes back on a networked machine.
- [ ] **M1-5 — Typed deserialization.** `serde` structs for the fields we show (name, downloads, likes). *Done when:* the response deserializes into typed structs.
- [ ] **M1-6 — Headless table output.** Print results as a plain table. Reuse `github/rust-hf-downloader/src/headless.rs`. Honor the global `--json` flag (M0-8): `search --json` emits the result list as JSON instead of the table. *Done when:* `search` prints a readable list, and `search --json` emits parseable JSON.
- [ ] **M1-7 — `tracing` + `--verbose`.** Init a subscriber; `--verbose` emits debug spans for the request. *Done when:* `search --verbose` shows debug output.
- [ ] **M1-8 — Error & offline handling.** Handle "no results", HTTP errors, and offline degradation with clear messages. *Done when:* each case prints a friendly message instead of panicking.
- [ ] **M1-9 — Mocked HTTP tests.** Use `wiremock` to test parsing against canned responses. *Done when:* `cargo test` covers success + error paths.

**Verify:** real results returned on any networked machine; handles "no results" and HTTP errors. CLI `--verbose` flag emits debug spans. Falls back gracefully if offline.

---

## M2 — `download` (async concurrency)

**Goal:** download model weights with progress, resume, and verification.

**Deliverable:** `vllm-deck download <model>` fetches files into a local cache.

**New concepts:** concurrent in-flight downloads (`futures`/`tokio` tasks) · streaming response bodies · progress reporting · resumable downloads (HTTP range requests) · checksum verification (`sha2`).

**Steps**

- [ ] **M2-1 — `Download` subcommand + cache path.** Add `Download { model: String }`; resolve a local cache directory. *Done when:* the command resolves where files will land.
- [ ] **M2-2 — List model files.** Query HF for the file manifest of the model. *Done when:* the command prints the files it would download.
- [ ] **M2-3 — Streaming single-file download.** Stream one response body to disk (not load-all-in-memory). Reuse `github/rust-hf-downloader/src/download.rs`. *Done when:* one file downloads to the cache.
- [ ] **M2-4 — Progress reporting.** Per-file progress (bytes / total). *Done when:* a progress indicator updates during download.
- [ ] **M2-5 — Concurrent downloads.** Fetch multiple files concurrently with bounded parallelism (`futures`/`tokio` tasks). *Done when:* multiple files download at once without overwhelming the host.
- [ ] **M2-6 — Resume via range requests.** On restart, detect partial files and continue with HTTP range requests. *Done when:* interrupting and re-running resumes rather than restarting.
- [ ] **M2-7 — Checksum verification.** Verify each file with `sha2` against HF metadata. *Done when:* a corrupted/partial file is detected and re-fetched.
- [ ] **M2-8 — Unit tests.** Cover chunk/range calculation and file hashing. *Done when:* `cargo test` passes.

**Verify:** a small model downloads; interrupting and re-running resumes rather than restarting. Unit tests for chunked calculation and file hashing.

---

## M2.5 — Workspace Refactor & CI/CD

**Goal:** Refactor the growing crate into a Cargo workspace and set up cross-arch CI.

**New concepts:** Cargo workspaces · library separation · GitHub Actions matrix builds · cross-compilation considerations.

**Steps**

- [ ] **M2.5-1 — Create the workspace.** Add a root workspace `Cargo.toml` listing member crates. *Done when:* `cargo build` builds the workspace.
- [ ] **M2.5-2 — Extract `core` library crate.** Move GPU/VRAM/API/download logic into a `core` lib crate; keep the binary as a thin CLI shell. *Done when:* the binary depends on `core` and still runs.
- [ ] **M2.5-3 — Re-point tests.** Move/adjust unit tests into the crate that owns the logic. *Done when:* `cargo test` passes across the workspace.
- [ ] **M2.5-4 — CI matrix.** Extend `ci.yml` to run `fmt`, `clippy`, `test` on `x86_64` and `aarch64`. *Done when:* CI is green on both targets.

**Deliverable:** A `core` library crate is extracted. CI runs `fmt`, `clippy`, and `test` on `x86_64` and `aarch64`.

---

## Pre-M3 — Resolve the aarch64 vLLM image (de-risking spike)

**Goal:** Before writing any Docker code, confirm a usable ARM64 vLLM image exists for the DGX Spark.

**Why first:** the stock `vllm/vllm-openai` image is x86_64-only. On the Spark (aarch64) M3 is blocked
unless a working ARM64 image is available (likely an NVIDIA NGC vLLM container). This is an external
unknown, not a coding task — resolve it *before* M3-1 rather than discovering a wall mid-milestone.

**Steps**

- [ ] **Pre-M3-1 — Identify & test the aarch64 image.** A concrete ARM64 image reference (registry + tag) is identified and `docker pull`-tested on the Spark — **or** a fallback is documented (build-from-source, or x86-only until an image lands). *Done when:* the image tag (or fallback) is recorded for M3-2 to consume.

---

## M3 — `deploy` (Docker Engine API)

**Goal:** pull and launch a vLLM container, wait until healthy.

**Deliverable:** `vllm-deck deploy <model>` runs vLLM with GPU passthrough on a local model.

**New concepts:** `bollard` (Docker Engine API client) · container create/start/stop/logs · GPU device passthrough · volume mounts + port mapping · readiness/health polling · `toml` config profiles (`~/.config/vllm-deck/`).

**Steps**

- [ ] **M3-1 — `bollard` connection.** Connect to the local Docker daemon and ping it. *Done when:* the command confirms it can talk to Docker.
- [ ] **M3-2 — Per-arch image selection.** Select the vLLM image per architecture: stock `vllm/vllm-openai` on x86, and on aarch64 the ARM64 image identified in the **Pre-M3 spike** for the DGX Spark. *Done when:* the right image tag is resolved per host arch.
- [ ] **M3-3 — Pull image.** Pull with progress. *Done when:* the image is present locally.
- [ ] **M3-4 — `Deploy` subcommand + container create.** Add `Deploy { model: String }`; create the container with volume mounts (model cache) and port mapping. *Done when:* a container is created (not yet started).
- [ ] **M3-4b — `--dry-run` flag.** `deploy --dry-run <model>` prints the exact image, container config, volume mounts, port mapping, and GPU passthrough flags that *would* be used — without pulling images or talking to the Docker daemon. Great for debugging and trust-building. *Done when:* `deploy --dry-run <model>` prints the full spec and exits 0 with no daemon calls.
- [ ] **M3-5 — GPU passthrough.** Configure GPU device passthrough in the container spec. *Done when:* the container sees the GPU.
- [ ] **M3-6 — Start + log streaming.** Start the container and stream its logs. *Done when:* `deploy` starts vLLM and shows its startup logs.
- [ ] **M3-7 — Health polling.** Poll readiness until the server is healthy (or time out with a clear error). *Done when:* `deploy` reports healthy once vLLM is serving.
- [ ] **M3-8 — `toml` config profiles.** Read/write profiles under `~/.config/vllm-deck/` (image, ports, defaults). *Done when:* a profile drives deploy settings.
- [ ] **M3-9 — Integration test.** Test the Docker workflow end to end (create → start → health → stop). *Done when:* the integration test passes.

**Verify:** container reaches healthy state on the ASUS 4060; repeat on the Spark with the ARM64 image. CLI integration tests verify Docker workflow.

---

## M4 — `monitor` / `dashboard` (TUI)

**Goal:** a real-time terminal dashboard of vLLM + GPU metrics.

**Deliverable:** `vllm-deck monitor` shows live panels; `dashboard` combines all views.

**New concepts:** `ratatui` widgets + layout · `crossterm` raw mode / event handling · an async event loop (input + a metrics-poll tick) · scraping vLLM's Prometheus `/metrics` (`prometheus-parse`) · live NVML stats · sparklines for history.

**Steps**

- [ ] **M4-1 — Terminal setup/teardown.** `crossterm` raw mode, alternate screen, and clean restore on exit/panic. *Done when:* entering and quitting the TUI leaves the terminal usable.
- [ ] **M4-2 — Static layout.** `ratatui` layout with empty panels (GPU, vLLM, history). *Done when:* a static dashboard renders.
- [ ] **M4-3 — Async event loop.** Combine an input stream and a metrics-poll tick; `q` quits. *Done when:* the loop handles input + ticks without blocking.
- [ ] **M4-4 — Live NVML panel.** Poll GPU stats and render them. Reuse `github/spark-dashboard/src/metrics/gpu.rs`. *Done when:* the GPU panel updates live.
- [ ] **M4-5 — vLLM Prometheus scrape.** Scrape `/metrics` and parse with `prometheus-parse`. Reuse `github/spark-dashboard/src/engines/vllm.rs`. *Done when:* the vLLM panel shows live serving metrics.
- [ ] **M4-6 — Sparklines / history.** Keep a rolling history buffer and render sparklines. *Done when:* trends render over time.
- [ ] **M4-7 — `dashboard` command.** Combine all panels into one view. *Done when:* `vllm-deck dashboard` shows the combined layout.
- [ ] **M4-8 — Widget rendering tests.** Test rendering with a `ratatui` test backend. *Done when:* rendering tests pass.

**Verify:** dashboard updates live against a running vLLM container on both Linux machines. Widget rendering tests pass.

---

## After M4 (polish)

- Better error messages, config profiles, demo GIF, publish to crates.io.
