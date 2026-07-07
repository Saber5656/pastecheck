# Research: Rust Crate Selection and Release Engineering Facts

Status: reviewed input for `docs/DESIGN.md` §4.3 (dependency policy) and §17
(release engineering). Versions verified against the crates.io API on
2026-07-08.

## 1. Runtime dependency candidates (verified current)

| Crate | Verified stable | Last update | Role in pastecheck | Notes |
|---|---|---|---|---|
| `unicode-security` | 0.1.2 | 2024-09-12 | TR39 skeleton (confusable detection), mixed-script resolution (`AugmentedScriptSet`) | unicode-rs org. Data lags latest Unicode versions — see caveat below |
| `unicode-script` | 0.5.8 | 2025-12-03 | Script property per char (homoglyph tokens) | unicode-rs org |
| `unicode-segmentation` | 1.13.3 | 2026-06-01 | Word segmentation for homoglyph tokenization | unicode-rs org |
| `unicode-properties` | 0.1.4 | 2025-10-30 | General_Category + Emoji properties (Extended_Pictographic) for invisible/emoji-context rules | unicode-rs org |
| `clap` (derive) | 4.6.1 | 2026-04-15 | CLI parsing | brings `anstream`/`anstyle` (color handling, `NO_COLOR`) |
| `serde` + `serde_json` | 1.x / 1.0.150 | 2026-05 | JSON report output | |
| `toml` | 1.1.2 | 2026-04-01 | Config file + test-corpus parsing | TOML basic strings support `\uXXXX`/`\UXXXXXXXX` escapes — lets the attack corpus be stored fully escaped (reviewable, no raw hostile bytes in the repo) |
| `thiserror` | 2.x | current | Error taxonomy | `anyhow` not needed in a small bin with a defined exit-code map |

Dev-dependencies: `assert_cmd` 2.2.2, `predicates`, `proptest` 1.11.0,
`cargo-fuzz` (tooling, not a dependency). CI tooling: `cargo-deny` 0.19.9.

Explicitly rejected:

| Candidate | Why rejected |
|---|---|
| `arboard` (native clipboard) | Pulls X11/Wayland native stacks; pastecheck only ever *reads* clipboard and can shell out to `pbpaste`/`wl-paste`/`xclip` with less attack/dependency surface (ADR-005) |
| `vte` (ANSI parser) | Full terminal-emulation parser; we need *detection with spans*, not emulation. A ~100-line explicit state machine per ECMA-48 is easier to audit and to specify for implementation agents (DESIGN §7.2) |
| `regex` on untrusted input | Not needed; all detectors are single-pass scanners. Avoids pathological-input performance questions by construction |
| `unicode-names2` (char names) | Large data table; we only need names for the codepoints in our own rule tables, which we already enumerate |
| `insta` (snapshot tests) | Plain golden files + an `UPDATE_GOLDEN=1` helper keep the dev-dependency tree smaller and the workflow more mechanical |

### Caveat: TR39 data freshness

`unicode-security` 0.1.2 bundles confusables data behind the current Unicode
release. Impact: very recent codepoints may be missing from skeleton mapping.
Mitigations: (a) our per-codepoint tables (invisible, bidi, fullwidth,
punctuation) are owned by pastecheck and independent of that crate; (b) the
JSON report and `pastecheck rules` expose the underlying data version
(DESIGN §8.7) so staleness is observable; (c) recorded as a known unknown in
ISSUE_PLAN — if the lag matters in practice, a vendored-data update becomes a
new issue.

## 2. GitHub Actions facts (verified)

- **arm64 Linux hosted runners are GA and free for public repositories**
  (labels `ubuntu-24.04-arm`, `ubuntu-22.04-arm`; GA announced 2025-08-07:
  https://github.blog/changelog/2025-08-07-arm64-hosted-runners-for-public-repositories-are-now-generally-available/).
  Consequence: `aarch64-unknown-linux-musl` release builds run natively — no
  `cross`, no QEMU, one less trusted tool.
- macOS runners: `macos-14`/`macos-15` are Apple Silicon (native
  `aarch64-apple-darwin`); `macos-13` is x86_64 (native `x86_64-apple-darwin`).
- Release target matrix (DESIGN §17): darwin x86_64 + aarch64, linux-musl
  x86_64 + aarch64 — all built natively on hosted runners.

## 3. Release tooling decision input

Options considered for producing GitHub Releases artifacts:

| Option | Assessment |
|---|---|
| `cargo-dist` / `dist` | Generates workflows + installers; powerful, but adds a generator layer between the repo and its release process, and the project has changed stewardship — an extra moving dependency for a security tool |
| Hand-rolled workflow | ~100 lines of explicit YAML: 4-target matrix, `cargo build --release`, tar + `sha256sum`, upload. Fully auditable, SHA-pinned actions, no tool trust. Chosen for v1 (ADR-006) |

Supply-chain posture for workflows (applies to CI and release):

- All third-party actions pinned to **full commit SHAs** (not tags).
- Default `permissions: {}` at workflow level; jobs opt in (`contents: read`;
  release job `contents: write`).
- `Cargo.lock` committed (binary crate); `cargo-deny` gates advisories,
  license allowlist (MIT/Apache-2.0 compatible), and duplicate/banned crates.
- Dependabot for `cargo` + `github-actions` ecosystems, weekly.
- crates.io publishing stays **manual** in v1 (maintainer-run `cargo publish`);
  no long-lived registry token in repo secrets. Procedure documented in
  `RELEASING.md` (issue 25), consistent with the repository owner's policy
  that credentials are handled by the human.

## 4. Toolchain pinning

- `rust-toolchain.toml` pins a concrete stable (starting point: the current
  stable at scaffolding time; issue 01 records the exact version).
- `Cargo.toml` sets `rust-version` (MSRV) = pinned stable at v1 start; CI has
  an MSRV check job. MSRV bumps are deliberate PRs, not side effects.
- `edition = "2024"`.
- `#![forbid(unsafe_code)]` at crate root; enforced by clippy config too.
