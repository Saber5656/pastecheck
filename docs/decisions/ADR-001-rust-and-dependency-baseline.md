# ADR-001: Rust with a minimal, allowlisted dependency baseline

Status: accepted (2026-07-08, confirmed with product owner)

## Context

pastecheck runs on every paste inside interactive shells (latency-sensitive),
parses hostile input (memory-safety-sensitive), and is distributed as an OSS
security tool (supply-chain-sensitive). Language candidates: Rust, Go, Python.

## Decision

Rust, edition 2024, `#![forbid(unsafe_code)]`, lib+bin crate, with a fixed
runtime dependency allowlist (DESIGN §4.3): `clap`, `serde`, `serde_json`,
`toml`, `thiserror`, `unicode-security`, `unicode-script`,
`unicode-segmentation`, `unicode-properties`. Adding a runtime dependency
requires a new ADR. `Cargo.lock` is committed; `rust-toolchain.toml` pins the
toolchain; MSRV is declared via `rust-version` and CI-checked.

## Alternatives considered

- **Go**: comparable binary/startup story, but no maintained TR39
  confusables/skeleton implementation — we would own security-critical
  Unicode data generation (verified in research 03). Rejected.
- **Python**: runtime dependency and interpreter startup in the per-paste hot
  path; unsuitable for the hook form factor. Rejected.
- **Rust with `vte`/`regex`/`arboard`**: rejected per research 03 §1 —
  hand-rolled bounded ANSI state machine, no regex on untrusted input,
  clipboard via system utilities (ADR-005).

## Consequences

- unicode-rs crates give TR39 skeletons, script sets, segmentation, and
  Extended_Pictographic without owning the data (staleness caveat tracked as
  a known unknown).
- Implementation issues can specify exact crate APIs, making them executable
  by low-capability agents.
- Slightly higher implementation effort than Go/Python; accepted.
