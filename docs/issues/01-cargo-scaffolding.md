# Title

Cargo project scaffolding, licenses, and lint policy

## Summary

Initialize the Rust crate (lib + bin) with the exact repository layout,
dual license files, toolchain pinning, and lint policy required by
DESIGN §4.2 and SEC-11, so every later issue lands on a fixed skeleton.

## Context

The repository currently contains only `README.md` and `docs/`. Every
subsequent issue assumes this skeleton exists. License decision: MIT OR
Apache-2.0 (product owner, 2026-07-08).

## Scope

- `Cargo.toml`, `src/lib.rs`, `src/main.rs`, module stubs, license files,
  `rust-toolchain.toml`, `.gitignore`, `rustfmt.toml` (only if non-default
  settings are needed — otherwise omit).
- No detector logic, no CLI parsing beyond a `--version` placeholder.

## Detailed Requirements

1. `Cargo.toml`:
   - `name = "pastecheck"`, `version = "0.1.0"`, `edition = "2024"`,
     `license = "MIT OR Apache-2.0"`, `repository`, `description`
     ("Terminal paste-safety guard: detects newlines, invisible characters,
     escape sequences, bidi overrides, and homoglyphs"),
     `rust-version` set to the pinned toolchain's version.
   - `[dependencies]`: none yet (added per-issue; ADR-001 allowlist governs).
   - `[lints.rust] unsafe_code = "forbid"`;
     `[lints.clippy] all = { level = "deny", priority = -1 }` plus
     `pedantic = "warn"`.
   - `[profile.release] strip = true, lto = "thin"`.
2. `src/lib.rs`: `#![forbid(unsafe_code)]`; declare empty module tree exactly
   as DESIGN §4.2 (`cli`, `config`, `input`, `engine`, `model`, `rules`,
   `detectors`, `sanitize`, `render`, `clipboard`, `shell`) with
   `pub mod`s and empty `mod.rs` files where directories are specified.
3. `src/main.rs`: prints `pastecheck <version>` for `--version`/`-V` (manual
   arg check, no clap yet) and exits 2 with a one-line usage hint otherwise.
4. `rust-toolchain.toml`: pin the current stable channel by explicit version
   (e.g. `channel = "1.xx.y"` — record the number chosen in the PR body).
5. `LICENSE-MIT` and `LICENSE-APACHE` with standard texts, copyright
   `pastecheck contributors`.
6. `.gitignore`: `/target`, `**/*.rs.bk`.
7. Commit `Cargo.lock`.

## Acceptance Criteria

- [ ] `cargo build` and `cargo test` succeed on a clean checkout.
- [ ] `cargo run -- --version` prints `pastecheck 0.1.0`; any other
      invocation exits with code 2.
- [ ] `cargo clippy --all-targets -- -D warnings` and `cargo fmt --check`
      pass.
- [ ] Both license files exist; `Cargo.toml` license field matches.
- [ ] Module tree compiles and matches DESIGN §4.2 names exactly.

## Validation

```sh
cargo build --locked && cargo test --locked
cargo run -q -- --version
cargo clippy --all-targets -- -D warnings && cargo fmt --check
test -f LICENSE-MIT && test -f LICENSE-APACHE && test -f Cargo.lock
```

## Dependencies

None (first issue).

## Non-goals

CI workflows (issue 02), cargo-deny (03), any subcommand behavior (13).

## Design References

- `docs/DESIGN.md` §4.2 (layout), §3.3 SEC-11, §17 (profile)
- `docs/decisions/ADR-001-rust-and-dependency-baseline.md`
