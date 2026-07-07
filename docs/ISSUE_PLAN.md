# pastecheck v1 Issue Plan

Status: canonical implementation plan derived from `docs/DESIGN.md`.
GitHub Issues are generated from `docs/issues/NN-*.md`; when they disagree
with these files, these files win and the GitHub Issues are stale.

## 1. v1 completion statement

v1 is complete when all 27 issues below are closed with their Acceptance
Criteria and Validation sections satisfied. At that point the repository
ships: a Rust binary `pastecheck` with `check` / `sanitize` / `gate` /
`shell-init` / `rules` subcommands implementing the 26-rule registry
(DESIGN §5.3) across 5 detector categories; escaped-safe human output, stable
JSON (schema v1), TOML config with per-rule overrides and allowlist;
clipboard read on macOS/Linux; zsh and bash paste guards with
paste/abort/sanitize prompts and the SEC-9 failure policy; an escaped attack
corpus with golden outputs, property tests, fuzz targets, and pty-driven
shell E2E tests; hardened CI (fmt/clippy/test/MSRV/cargo-deny, SHA-pinned
actions) and a tag-driven release workflow producing binaries + SHA256SUMS
for 4 targets; README/SECURITY/CONTRIBUTING/RELEASING/CHANGELOG. No feature
of DESIGN v1 scope exists outside this issue list.

## 2. Issue list (recommended execution order)

| # | File | Title | Wave |
|---|---|---|---|
| 01 | `issues/01-cargo-scaffolding.md` | Cargo project scaffolding, licenses, lint policy | 0 |
| 02 | `issues/02-ci-build-test-lint.md` | CI workflow: fmt, clippy, test matrix, MSRV | 0 |
| 03 | `issues/03-supply-chain-ci.md` | cargo-deny gate and Dependabot | 0 |
| 04 | `issues/04-core-data-model.md` | Core data model and rule registry | 1 |
| 05 | `issues/05-detector-engine.md` | Detector trait, FindingsBuilder, engine pipeline | 1 |
| 06 | `issues/06-input-acquisition.md` | Input acquisition: sources, size cap, UTF-8 policy (PC220) | 1 |
| 07 | `issues/07-detector-newline.md` | Newline detector (PC101–PC105) | 2 |
| 08 | `issues/08-ansi-escape-parser.md` | ANSI escape parser detector (PC201–PC205) | 2 |
| 09 | `issues/09-detector-control-residual.md` | Residual control detector (PC206, PC210–PC212) | 2 |
| 10 | `issues/10-detector-invisible.md` | Invisible detector with emoji/IVS context (PC301–PC304) | 2 |
| 11 | `issues/11-detector-bidi.md` | Bidi detector (PC401–PC403) | 2 |
| 12 | `issues/12-detector-homoglyph.md` | Homoglyph detector (PC501–PC504) | 2 |
| 13 | `issues/13-cli-skeleton.md` | CLI command tree, exit codes, `rules` subcommand | 3 |
| 14 | `issues/14-escaping-module.md` | Output escaping module (SEC-3) | 3 |
| 15 | `issues/15-human-renderer.md` | Human report renderer | 3 |
| 16 | `issues/16-json-output.md` | JSON output (schema v1) | 3 |
| 17 | `issues/17-sanitizer.md` | Sanitizer engine and `sanitize` subcommand | 3 |
| 18 | `issues/18-config-loading.md` | TOML config loading, precedence, validation | 3 |
| 19 | `issues/19-clipboard-adapter.md` | Clipboard adapters (`--clipboard`) | 4 |
| 20 | `issues/20-gate-subcommand.md` | `gate` subcommand (hook protocol) | 4 |
| 21 | `issues/21-zsh-hook.md` | zsh hook snippet and `shell-init zsh` | 4 |
| 22 | `issues/22-bash-hook.md` | bash hook snippet (intercept + keybind modes) | 4 |
| 23 | `issues/23-shell-e2e-harness.md` | pty-driven shell E2E test harness | 4 |
| 24 | `issues/24-attack-corpus.md` | Escaped attack corpus and golden tests | 5 |
| 25 | `issues/25-fuzzing.md` | Fuzz targets and scheduled fuzz CI | 5 |
| 26 | `issues/26-release-workflow.md` | Release workflow (4 targets) and RELEASING.md | 5 |
| 27 | `issues/27-user-docs.md` | README, SECURITY.md, CONTRIBUTING.md, CHANGELOG | 5 |

## 3. Dependency table

| Issue | Blocked by | Notes |
|---|---|---|
| 01 | — | |
| 02 | 01 | |
| 03 | 01 | |
| 04 | 01 | |
| 05 | 04 | |
| 06 | 04 | PC220 emission lives here |
| 07 | 05 | |
| 08 | 05 | |
| 09 | 05, 08 | scans chars not consumed by 08 |
| 10 | 05 | |
| 11 | 05 | |
| 12 | 05 | |
| 13 | 05, 06 | |
| 14 | 04 | table names filled in progressively by 07–12 |
| 15 | 13, 14 | |
| 16 | 04, 13 | |
| 17 | 05, 07–12 | strategies need all rule spans |
| 18 | 04, 13 | |
| 19 | 13 | |
| 20 | 13, 14 | |
| 21 | 17, 20 | sanitize path |
| 22 | 17, 20 | |
| 23 | 21, 22 | |
| 24 | 07–12, 15, 16 | golden outputs need renderers |
| 25 | 05–12, 17 | |
| 26 | 02 | |
| 27 | 18, 20, 21, 22 | documents final UX |

## 4. Implementation waves

- **Wave 0 — foundations**: 01 → {02, 03} (parallel). Repo builds, CI green,
  supply-chain gates on from the first PR.
- **Wave 1 — core**: 04 → {05, 06} (parallel). Types, registry, pipeline,
  input policy.
- **Wave 2 — detectors**: {07, 08→09, 10, 11, 12} all parallel after 05.
  Each detector is one issue, one module, one PR.
- **Wave 3 — CLI surface**: 13 & 14 (parallel) → {15, 16, 18} (parallel),
  17 after detectors. End of wave: `pastecheck check/sanitize/rules` fully
  usable via stdin/file.
- **Wave 4 — clipboard & hooks**: {19, 20} (parallel) → {21, 22} (parallel)
  → 23. End of wave: guard works in zsh and bash.
- **Wave 5 — hardening & shipping**: {24, 25, 26} (parallel) → 27.

Single-implementer order: 01 02 03 04 05 06 08 09 07 10 11 12 13 14 15 16 18
17 19 20 21 22 23 24 25 26 27.

## 5. Coverage table (DESIGN.md § → issues)

| DESIGN section | Issue(s) |
|---|---|
| §1–2 overview/scope | plan-level (this file) |
| §3.2 boundaries B1–B7 | B1: 06 · B2: 19 · B3: 18 · B4: 13 · B5: 14, 15, 16, 20 · B6: 20, 21, 22 · B7: 02, 03, 26 |
| §3.3 SEC-1 | 01 (dependency policy), 03 |
| §3.3 SEC-2 | 06, 13, 15 (error/report content rules) |
| §3.3 SEC-3 | 14 (module), 24 (product-level property test) |
| §3.3 SEC-4 | 05 (caps), 06 (size), 08 (sequence cap), 15/20 (output caps) |
| §3.3 SEC-5 | 06 |
| §3.3 SEC-6, SEC-7 | 19 |
| §3.3 SEC-8, SEC-9 | 20, 21, 22 (verified in 23) |
| §3.3 SEC-10 | 18, 20 |
| §3.3 SEC-11 | 01, 02 |
| §3.3 SEC-12 | 02, 03 |
| §3.3 SEC-13 | 26 |
| §3.3 SEC-14 | 13 |
| §3.3 SEC-15, SEC-16 | 27 |
| §4.2 layout / §4.4 engine | 01, 05 |
| §5 data model & registry | 04 |
| §6 input | 06 |
| §7.1 | 07 |
| §7.2 | 08, 09 |
| §7.3 | 10 |
| §7.4 | 11 |
| §7.5 | 12 |
| §8.1–8.3, §8.7 exit codes & tree | 13 (+16 for `rules --format json`) |
| §8.4 | 17 |
| §8.5 | 20 |
| §8.6 | 13 (dispatch), 21, 22 (snippets) |
| §9.1 | 14 |
| §9.2 | 15 |
| §9.3 | 16 |
| §10 | 17 |
| §11 | 18 |
| §12 | 19 |
| §13.1 | 21 |
| §13.2 | 22 |
| §13.3 | 20, 21, 22, 23 |
| §14 | 06, 13 |
| §15 | 20, 24 (timing tests) |
| §16 | 02 (CI), 23, 24, 25 + per-issue Validation |
| §17 | 03, 26 |
| §18 | 26 (RELEASING), 27 |

Maintenance rule: any DESIGN change updates this table in the same PR.

## 6. Product-level validation strategy

| Layer | Gate | Where |
|---|---|---|
| Static | `cargo fmt --check`, `clippy -D warnings`, `forbid(unsafe_code)`, MSRV check | every PR (02) |
| Supply chain | cargo-deny advisories/licenses/bans/sources | every PR (03) |
| Unit + property | per-module tests; sanitizer §10.3 and escaping SEC-3 properties | every PR (each issue) |
| Product corpus | escaped attack corpus + golden human/JSON outputs; renderer output-safety assertion | every PR once 24 lands |
| CLI contract | assert_cmd integration: exit codes, sources, config precedence | every PR once 13 lands |
| Hook E2E | expect/pty scenarios on real zsh + bash (ubuntu), zsh (macos) | every PR once 23 lands |
| Fuzz | 3 targets, corpus-seeded, weekly scheduled, non-blocking but triaged | 25 |
| Performance | `#[ignore]` timing tests vs §15 budgets, nightly non-blocking | 24 |
| Release | `--locked` builds from clean checkout; SHA256SUMS verified in workflow | 26 |

Definition of done for every issue: code + tests merged, CI green, and the
issue's Validation commands pass locally as written.

## 7. Deferred to v2 (do not implement in v1)

Suspicious visible whitespace detector (NBSP U+00A0, thin spaces, U+205F…);
dangerous-command content heuristics; fish shell hook; Windows support;
Homebrew tap (needs separate `homebrew-tap` repo — out of this repo's scope);
`sanitize --write-clipboard`; punycode/IDN URL analysis; private-use and
unassigned codepoint flagging; localized (Japanese) messages; sigstore /
build provenance attestation; streaming large-file mode; vendored Unicode
data updates; `dist`-based installer generation.

## 8. Known unknowns (may spawn new issues during implementation)

| # | Unknown | Contingency |
|---|---|---|
| U1 | bash `bind -x` interception robustness across readline builds/distros (research 02 §3) | issue 22 ships dual modes; if intercept mode proves unreliable on a mainstream distro, demote it to opt-in and make `keybind` the default → follow-up issue |
| U2 | zsh plugin conflicts (bracketed-paste-magic, safe-paste) beyond the warn-and-replace policy | field reports → compatibility shim issue |
| U3 | `unicode-security` TR39 data lag vs current Unicode | if a real-world confusable is missed, vendor updated confusables data → new issue |
| U4 | expect/pty flakiness in CI (23) | quarantine job, keep unit-tested pure shell functions as the gate, document manual checklist → follow-up issue if flake rate > ~1% |
| U5 | Wayland clipboard tool variance (wl-paste flags/behavior across compositors) | adapter matrix expansion in 19 → follow-up issue |
| U6 | `zle -M` rendering limits for multi-line reports on small terminals | cap lines (already in §8.5); if unreadable in practice, add pager-style details mode → follow-up issue |
| U7 | Homoglyph false-positive/negative tuning of `ascii_ratio_threshold` | constant is config-exposed; corpus grows with reported cases |

## 9. Out-of-repo work recorded as handoffs (not issues here)

- Homebrew tap repository (`Saber5656/homebrew-tap`) — v2, separate repo.
- Enabling GitHub private vulnerability reporting and branch-protection
  settings are repository-settings actions performed by the owner (referenced
  by issue 27 / SEC-16); the docs specify *what* to enable.
