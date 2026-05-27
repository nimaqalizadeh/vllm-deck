# ADR-0002: Re-sequence the build; introduce async at M1, not week 1

- **Status:** accepted
- **Date:** 2026-05-29

## Context

The README's original phases (CLI skeleton → HF → Docker → TUI) put four of the hardest Rust topics —
async networking, NVML/FFI, the Docker API, and a full-screen TUI — into the first week or two. The
owner is a Rust beginner whose explicit goals are (1) learn Rust well and (2) build a resume-worthy
tool. The owner also explicitly wants to learn async, but was undecided on sequencing.

## Decision

Re-sequence into milestones M0–M4 (see [roadmap](../roadmap.md)) so each hard concept arrives alone,
and always leaves a runnable binary. **Async is introduced at M1** (`search`), where a single HTTP
request gives it a concrete, motivated reason to exist — not at M0.

## Rationale & trade-offs

- **For:** one new hard concept at a time prevents the "everything is unfamiliar at once" wall;
  async lands with motivation (you can *see* why `.await` beats blocking) rather than as cargo-cult;
  a working binary at every step keeps momentum and is demoable.
- **Against:** slightly slower to the most impressive feature (the TUI is last). Accepted, because the
  goal weights learning equally with the demo, and the TUI is far easier to build well once async and
  the data layers already work.
- **Rejected alternative — "resume-impact first" (TUI early):** maximally impressive but stacks
  async + ratatui + NVML in week one; highest risk of stalling.
- **Rejected alternative — README order as-is:** touches async and FFI simultaneously in phase 1.

## Consequences

`main` is synchronous through M0, then becomes `#[tokio::main]` from M1 onward. Dependencies are added
per milestone rather than all upfront.
