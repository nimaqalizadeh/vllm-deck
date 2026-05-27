# ADR-0001: Start as a single binary crate, refactor to a workspace later

- **Status:** accepted
- **Date:** 2026-05-29

## Context

The end-state design wants shared logic (GPU detection, memory math, HF client) reused by both the CLI
and the TUI. The reference project `github/llmfit` solves this with a Cargo **workspace**: a
`llmfit-core` library crate plus thin frontend crates (`llmfit-tui`, `llmfit-desktop`, ...). A
workspace is a set of related crates built together that can depend on each other.

## Decision

Start vllm-deck as a **single binary crate**. Defer the split into a `core` library + binary until
there is a second consumer of the shared logic (the TUI at M4), at which point the duplication or
sharing need is concrete.

## Rationale & trade-offs

- **For:** less ceremony now; no `pub`-everything API surface to design before we understand the
  domain; faster iteration; the eventual extraction is itself a high-value learning exercise (you'll
  *feel* why the boundary belongs where it does).
- **Against:** a later refactor touches imports across the codebase. Mitigated by keeping layers
  already separated into modules (`gpu/`, `hf/`, ...) so the extraction is mostly mechanical.
- **Rejected alternative:** start with the full llmfit-style workspace. Rejected because designing a
  library API before the first feature exists is premature abstraction for a Rust beginner.

## Consequences

Modules in the binary stay plain `mod` (private) until extraction; at that point exposed items become
`pub` and move into `vllm-deck-core`.
