# pastecheck v1 Design

Status: canonical design for v1. Derived issue plan: `docs/ISSUE_PLAN.md`.
Decisions with alternatives are recorded in `docs/decisions/ADR-*.md`.
Research background: `docs/research/*.md`.

Requirement decisions confirmed with the product owner on 2026-07-08:
CLI core + shell hooks (zsh, bash) / macOS + Linux / Rust /
report + interactive confirm + opt-in sanitize / detection categories =
newline, control+ANSI, invisible, bidi, homoglyph / license MIT OR Apache-2.0 /
distribution GitHub Releases + crates.io.

---

## 1. Product overview

**pastecheck** is a terminal paste-safety guard. It analyzes text that is
about to enter a terminal and reports characters and sequences that make the
text behave differently from how it looks: line breaks that execute
immediately, ANSI/control sequences that manipulate the terminal, invisible
characters, bidirectional-override reordering (Trojan Source), and
homoglyph/confusable substitution.

It ships as:

1. A single static Rust binary `pastecheck` with subcommands `check`
   (analyze), `sanitize` (emit cleaned text), `gate` (hook-facing protocol),
   `shell-init` (emit shell snippets), and `rules` (rule introspection).
2. Thin zsh and bash snippets (emitted by `shell-init`) that intercept
   bracketed paste at the prompt, call `pastecheck gate`, and ask the user to
   continue / abort / paste-sanitized when findings exist.

Primary usage:

```sh
# explicit inspection
pbpaste | pastecheck
pastecheck --clipboard
pastecheck suspicious.txt

# cleaned output (never in-place, never to clipboard)
pbpaste | pastecheck sanitize

# always-on guard
echo 'eval "$(pastecheck shell-init zsh)"' >> ~/.zshrc
echo 'eval "$(pastecheck shell-init bash)"' >> ~/.bashrc
```

Design priorities, in order: (1) never make the user less safe than no tool
(the tool's own output and failure modes are part of the threat model);
(2) near-zero false alarms for everyday CJK/emoji text; (3) low latency in the
paste path; (4) mechanical implementability — every behavior in this document
is specified so a low-capability implementation agent can execute it without
guessing.

## 2. Scope

### 2.1 v1 goals

| # | Goal |
|---|---|
| G1 | Detect the 5 categories with the exact rule set of §5.3 on macOS and Linux |
| G2 | Human-readable report that is itself safe to display (§9.1) |
| G3 | Stable JSON output (`--format json`, schema v1) |
| G4 | Opt-in sanitizer with per-category strategies (§10) |
| G5 | Clipboard read source (`--clipboard`) on macOS and Linux |
| G6 | zsh and bash paste guards with continue/abort/sanitize prompt |
| G7 | TOML config with per-rule severity override and allowlist |
| G8 | Attack corpus, property tests, fuzz targets, shell E2E tests |
| G9 | Hardened CI and tag-driven release binaries for 4 targets + checksums |

### 2.2 v1 non-goals

- No clipboard **writes** (no `pbcopy`, no OSC 52 emission) — ADR-005.
- No daemon / clipboard watcher; no GUI; no editor plugins.
- No Windows support.
- No content-level command heuristics (`curl | sh`, `rm -rf`).
- No network I/O of any kind (also a security requirement, SEC-1).
- No auto-sanitize default: pastecheck never modifies text unless asked.

### 2.3 Deferred to v2 (recorded, not designed here)

Suspicious visible whitespace (NBSP, thin spaces); fish shell; Windows;
Homebrew tap; `sanitize --write-clipboard`; dangerous-command heuristics;
private-use/unassigned codepoint flagging; punycode/IDN URL analysis;
localized (Japanese) report messages; sigstore/provenance attestation;
streaming mode for large files; Unicode data vendoring.

### 2.4 Known unknowns

Tracked in `docs/ISSUE_PLAN.md` §8; the material ones: bash interception
robustness across readline builds; zsh plugin-conflict matrix;
`unicode-security` data freshness; CI pty test flakiness; Wayland clipboard
tool variance.

## 3. Security model

### 3.1 Assets and attacker model

Assets: the user's shell session (ability to execute commands), the system
clipboard, the user's trust in what they read on screen, and the scanned
content itself (may be a secret — e.g. a pasted token).

Attacker: controls arbitrary content that the user copies (web page, README,
chat, PDF, email, AI output). The attacker does not control the user's machine,
config file, or the pastecheck binary. Copy-time substitution (pastejacking)
is assumed possible, so "the user read the text before copying" provides no
protection. See `docs/research/01-terminal-paste-threats.md`.

### 3.2 Trust boundaries

| # | Boundary | Direction | Trust level | Controls |
|---|---|---|---|---|
| B1 | stdin / FILE input | in | hostile | size cap, UTF-8 policy, linear-time scanning (SEC-4, SEC-5) |
| B2 | Clipboard via system utilities | in | hostile content, semi-trusted tool | fixed argv allowlist, timeout, output cap (SEC-7) |
| B3 | Config file | in | user-trusted, parsed defensively | data-only TOML, loud unknown-key warnings, gate falls back to defaults on parse failure (SEC-10) |
| B4 | CLI arguments | in | user-trusted | clap validation; content never accepted via argv |
| B5 | Terminal output (report, gate text) | out | **hostile-adjacent**: output derived from hostile input is re-displayed on a terminal | output escaping allowlist (SEC-3), bounded size (SEC-4) |
| B6 | Shell hook ↔ binary | both | snippet handles hostile bytes inside the interactive shell | stdin-only transfer, no eval, no temp files, fail policy (SEC-8, SEC-9) |
| B7 | CI / release pipeline | out | public artifact integrity | SHA-pinned actions, least-privilege tokens, checksums, cargo-deny (SEC-12, SEC-13) |

### 3.3 Security requirements (normative)

Every SEC item is testable and referenced by issues; implementation agents may
not weaken them.

| ID | Requirement |
|---|---|
| SEC-1 | No network I/O at runtime. No telemetry, no update checks. No dependency that opens sockets. |
| SEC-2 | No persistence of scanned content: no temp files, no cache, no logging of input. Error messages must not embed input content (offsets, counts, and codepoint numbers of findings are allowed; reports are allowed to show *escaped* findings). |
| SEC-3 | All output derived from input (human report, gate text, JSON string values) is escaped per §9.1. The emitted byte stream never contains: C0 controls other than structural `\n` (and `\t` only in JSON-escaped form), DEL, C1, raw `ESC`, bidi controls, or any codepoint from the §7.3 invisible tables. Property-tested (issue 23). |
| SEC-4 | Bounded resources: input size cap before analysis (default 5 MiB, exit 3 when exceeded); all detectors O(n) single-pass; findings aggregated per rule code; ≤ 20 spans and ≤ 10 distinct offenders recorded per finding; human/gate output line-capped. No regex over untrusted input. |
| SEC-5 | Input is untrusted bytes. Invalid UTF-8 is never silently repaired: it raises PC220 and analysis continues on the lossy-decoded text (§6). NUL is a finding, not a crash. |
| SEC-6 | Clipboard is read-only. v1 never writes the clipboard by any mechanism. |
| SEC-7 | Subprocess allowlist: exactly `pbpaste`, `wl-paste`, `xclip`, `xsel` with fixed argument vectors (§12), resolved via PATH, executed without a shell, stdout capped at max_bytes+1, killed after 5 s timeout. |
| SEC-8 | Shell snippets never `eval` pasted content, never pass it via argv or environment, never write it to disk; transfer is stdin pipe only. Widget/handler code paths must be safe for arbitrary bytes. |
| SEC-9 | Hook failure policy (§13.3): scan-impossible (exit 3) → explicit user prompt, never silent pass-through; binary missing/crash → fail-open **with a visible warning** by default, `PASTECHECK_GATE_FAIL=closed` opt-in. Rationale in ADR-003. |
| SEC-10 | Config cannot execute code and cannot silently disable safety: unknown keys warn on stderr; a config that fails to parse causes `check`/`sanitize` to exit 3, but `gate` to continue with built-in defaults plus a warning (a broken config must not brick paste). |
| SEC-11 | `#![forbid(unsafe_code)]`; CI enforces `cargo clippy -- -D warnings` and `cargo fmt --check`. |
| SEC-12 | Supply chain: `Cargo.lock` committed; `cargo-deny` (advisories, license allowlist, bans, sources) gating CI; all GitHub Actions pinned by full commit SHA; workflows default `permissions: {}` with per-job opt-in; Dependabot for cargo + actions. |
| SEC-13 | Releases: tag-driven workflow builds from a clean checkout, artifacts named `pastecheck-<version>-<target>.tar.gz` with a `SHA256SUMS` file; release job is the only job with `contents: write`. |
| SEC-14 | Exit codes (§8.2) reliably distinguish clean / findings / usage error / operational error so scripts and hooks can fail safe. |
| SEC-15 | Documented residual risks: bracketed-paste end-marker truncation (research 02 §1); bash NUL fidelity; local-hook-vs-remote-shell gap over ssh. These must appear in user docs, not only here. |
| SEC-16 | `SECURITY.md` with GitHub private vulnerability reporting enabled and a response-time statement. |

### 3.4 Abuse cases (must-hold behaviors)

| Abuse | Required behavior |
|---|---|
| 5 MiB of `ESC[` bytes pasted | size cap or aggregated findings; report stays ≤ caps; no hang (SEC-4) |
| Paste containing `ESC[201~` early-terminator | captured prefix scanned; behavior documented (SEC-15) |
| Report content itself carrying an attack (e.g. context snippet includes OSC 52) | escaping guarantees inert output (SEC-3) |
| Clipboard utility replaced by a hostile binary on PATH | out of scope (attacker already executes code as user), but fixed argv + no shell prevents argument-injection amplification (SEC-7) |
| Config file points sanitize to "keep everything" then user relies on sanitize | `sanitize` prints a per-category summary of what was and was not removed (§10.4) |
| Hook binary deleted / broken install | visible warning + fail-open default; `PASTECHECK_GATE_FAIL=closed` for strict users (SEC-9) |
| Secrets in scanned text | no persistence, no network, no logs (SEC-1, SEC-2) |

## 4. Architecture

### 4.1 Components and data flow

```
                 ┌────────────────────────────────────────────────┐
 stdin/FILE ────▶│ input::acquire ──▶ engine::analyze ──▶ Report  │
 clipboard ─────▶│   (B1/B2)            │ detectors (§7)          │
                 │                      ▼                          │
                 │        render::human / render::json (§9)       │
                 │        sanitize::apply (§10)                   │
                 └────────────────────────────────────────────────┘
                        ▲ gate protocol (§8.5)         │ escaped text
              zsh/bash snippet (§13) ──────────────────┘
```

One process per invocation; no state between runs. The shell snippets contain
no analysis logic (ADR-003).

### 4.2 Repository and module layout

```
Cargo.toml  Cargo.lock  rust-toolchain.toml  deny.toml
LICENSE-MIT  LICENSE-APACHE  README.md  SECURITY.md  CHANGELOG.md  RELEASING.md
.github/workflows/ci.yml  .github/workflows/release.yml  .github/dependabot.yml
src/
  main.rs            # exit-code mapping only
  cli.rs             # clap command tree (§8.1)
  config.rs          # §11
  input.rs           # §6
  engine.rs          # detector pipeline (§4.4)
  model.rs           # §5.1
  rules.rs           # static rule registry (§5.3) + codepoint tables (§7)
  detectors/
    mod.rs           # Detector trait, registry
    newline.rs  control.rs  invisible.rs  bidi.rs  homoglyph.rs
  sanitize.rs        # §10
  render/
    mod.rs  human.rs  json.rs   # §9 (escape module lives in render/mod.rs)
  clipboard.rs       # §12
  shell/
    mod.rs           # shell-init subcommand (include_str! of snippets)
    snippets/pastecheck.zsh
    snippets/pastecheck.bash
tests/
  cli.rs             # assert_cmd integration tests
  corpus.rs          # corpus runner + golden comparison
  corpus/*.toml      # escaped attack corpus (§16.3)
  golden/*.txt       # expected human/json renderings
  e2e/*.exp          # expect scripts for zsh/bash pty tests
fuzz/fuzz_targets/{engine.rs,ansi.rs,sanitize.rs}
docs/                # this design, ADRs, research, issue plan
```

The crate is `lib + bin` (`src/lib.rs` re-exports modules) so unit and
property tests link the library directly.

### 4.3 Dependency policy

Runtime dependencies are limited to the allowlist below (rationale and
verified versions: `docs/research/03-rust-crates-and-release.md`). Adding a
runtime dependency requires an ADR.

`clap` (derive), `serde`, `serde_json`, `toml`, `thiserror`,
`unicode-security`, `unicode-script`, `unicode-segmentation`,
`unicode-properties`, plus clap's own `anstream`/`anstyle` for color.

Dev-only: `assert_cmd`, `predicates`, `proptest`. Tooling: `cargo-deny`,
`cargo-fuzz`.

### 4.4 Engine

```rust
pub trait Detector {
    fn category(&self) -> Category;
    fn scan(&self, input: &ScanInput<'_>, cfg: &EffectiveConfig, out: &mut FindingsBuilder);
}

pub struct ScanInput<'a> {
    pub text: &'a str,        // lossy-decoded when input had invalid UTF-8
    pub raw_bytes: usize,     // original byte length
    pub had_invalid_utf8: bool,
}
```

- Fixed execution order: `newline`, `control`, `invisible`, `bidi`,
  `homoglyph` (deterministic output; PC220 is emitted by `input`, not a
  detector).
- `FindingsBuilder` centralizes: aggregation by rule code (one `Finding` per
  code per run), span cap (20) and offender cap (10) (SEC-4), per-rule
  severity override and `off` (§11), allowlist suppression (§11), and final
  ordering (by first span start byte, then rule code).
- Detectors therefore only report occurrences; policy lives in one place.

## 5. Core data model

### 5.1 Types (normative shapes)

```rust
pub enum Category { Newline, Control, Invisible, Bidi, Homoglyph }

#[derive(PartialOrd, Ord)]           // Info < Warning < Critical
pub enum Severity { Info, Warning, Critical }

pub struct Span {                     // offsets into the analyzed text (§6)
    pub start_byte: usize,            // inclusive
    pub end_byte: usize,              // exclusive
    pub line: u32,                    // 1-based, split on LF only
    pub column: u32,                  // 1-based, in chars from line start
}

pub enum Offender { Codepoint(u32), RawByte(u8) }   // RawByte only for PC220

pub struct Finding {
    pub code: &'static str,           // "PC101" …
    pub category: Category,
    pub severity: Severity,           // effective (after config override)
    pub count: u32,                   // total occurrences in input
    pub spans: Vec<Span>,             // first ≤ 20, ascending
    pub spans_truncated: bool,
    pub offenders: Vec<Offender>,     // distinct, ≤ 10
    pub message: String,              // one line, from rule registry (+ detail)
    pub suggestion: Option<&'static str>, // what sanitize would do
}

pub struct Stats { pub bytes: usize, pub chars: usize, pub lines: u32 }

pub struct Report {
    pub findings: Vec<Finding>,       // ordered (§4.4)
    pub stats: Stats,
    pub max_severity: Option<Severity>,
}
```

### 5.2 Verdict semantics

- `clean` ⇔ no finding with severity ≥ `fail_on` (default `warning`).
  Info-level findings alone still yield exit 0 and verdict `clean`
  (they appear in JSON always, in human output only with `--show-info`).
- `max_severity` = maximum effective severity across all findings, or absent.

### 5.3 Rule registry (normative)

| Code | Category | Default severity | Name / condition |
|---|---|---|---|
| PC101 | newline | warning | LF (U+000A) occurrences that are internal (not the final EOL) |
| PC102 | newline | critical | Lone CR (U+000D not followed by LF): acts as Enter and overwrites the display |
| PC103 | newline | warning | CRLF pairs (internal) |
| PC104 | newline | warning | Line-break lookalikes: U+0085 NEL, U+000B VT, U+000C FF, U+2028 LS, U+2029 PS |
| PC105 | newline | critical | Trailing EOL: input ends with LF, CR, or CRLF — last line executes with no review |
| PC201 | control | critical | CSI sequence (`ESC [` … final byte) |
| PC202 | control | critical | OSC sequence other than OSC 52 |
| PC203 | control | critical | OSC 52 (clipboard write/read request) |
| PC204 | control | critical | DCS / SOS / PM / APC string |
| PC205 | control | critical | Bare, malformed, or unterminated ESC |
| PC206 | control | critical | C1 control character (U+0080–U+009F) |
| PC210 | control | critical | Other C0 control (incl. NUL, BS, BEL; excl. TAB/LF/CR) |
| PC211 | control | warning | DEL (U+007F) |
| PC212 | control | info | TAB (U+0009) — triggers completion in unbracketed pastes |
| PC220 | control | critical | Invalid UTF-8 byte sequence(s) in input |
| PC301 | invisible | critical | Invisible/format character (table §7.3.1) outside emoji/IVS context |
| PC302 | invisible | critical | Unicode tag characters U+E0000–U+E007F (ASCII smuggling) |
| PC303 | invisible | warning | Variation selector outside a recognized emoji/IVS context; runs of ≥ 4 escalate to critical |
| PC304 | invisible | info | ZWJ / VS15 / VS16 / IVS within a recognized emoji or ideographic variation context |
| PC401 | bidi | critical | Explicit overrides/embeddings: U+202A–U+202E |
| PC402 | bidi | critical | Directional isolates: U+2066–U+2069 |
| PC403 | bidi | warning | Directional marks: U+200E LRM, U+200F RLM, U+061C ALM |
| PC501 | homoglyph | critical | Mixed-script word whose TR39 skeleton is all-ASCII |
| PC502 | homoglyph | warning | Single-script non-Latin word, skeleton all-ASCII, in ASCII-majority input |
| PC503 | homoglyph | warning | Fullwidth ASCII variants: U+FF01–U+FF5E, U+3000 |
| PC504 | homoglyph | warning | Lookalike punctuation (table §7.5.4) |

Registry implementation: a single static table in `rules.rs`
(`code, category, default_severity, name, message, suggestion`) used by
detectors, renderers, `pastecheck rules`, and config validation. Adding a rule
means adding one row plus detector logic.

## 6. Input acquisition (§`input.rs`)

1. Sources, mutually exclusive: stdin (default when not a TTY or when `-`),
   `FILE` positional, `--clipboard` (§12). If stdin is a TTY and no source is
   given: usage error (exit 2) with a hint.
2. Read raw bytes with a hard cap of `max_bytes` (default 5 MiB). If the
   source yields more than `max_bytes`, stop reading, do **not** analyze,
   exit 3 with `input exceeds limit (<n> bytes > <max> bytes); not scanned`
   (SEC-4). The gate treats exit 3 as "could not scan" (§13.3).
3. UTF-8 policy (SEC-5): validate. If invalid sequences exist, record each
   maximal invalid run (byte offsets, offending bytes) → one aggregated PC220
   finding; decode with `String::from_utf8_lossy` and continue. All detector
   spans then refer to the **lossy-decoded text**; `Report.stats.bytes` is the
   original byte count. This dual-space caveat is stated in JSON docs.
4. Line/column mapping: precompute LF offsets once; `line` = 1 + count of LF
   before `start_byte`; `column` = 1 + char count from line start.
5. Empty input is valid: verdict clean, exit 0.

## 7. Detector specifications

Shared rules: single forward pass per detector; adjacent same-code occurrences
(e.g. a run of ZWSP) merge into one span; every table below lives in
`rules.rs` as `const` arrays with unit tests asserting exact membership.

### 7.1 `newline` (PC101–PC105)

Scan chars:

- Determine trailing EOL first: input ending in `\r\n` → one PC105 occurrence
  (span covers both bytes); ending in `\n` or `\r` → one PC105 occurrence.
  The trailing EOL is excluded from PC101/102/103 counting (no double
  report).
- Remaining `\r\n` pairs → PC103 (span per pair). Remaining `\n` → PC101.
  Remaining `\r` → PC102.
- U+0085, U+000B, U+000C, U+2028, U+2029 → PC104.

Message detail: PC105's message names the exact form (`LF`/`CR`/`CRLF`).
Suggestions: PC101/103/104/105 → "review before pasting; sanitize keeps
newlines by default (§10)"; PC102 → "remove".

### 7.2 `control` (PC201–PC212; PC220 comes from input)

Two-layer scan:

**Layer 1 — ANSI sequence parser** (explicit state machine; no regex, SEC-4).
States: `Ground`, `Esc`, `Csi`, `OscString`, `DcsLikeString(kind)`,
each with a per-sequence length cap of 4096 chars.

| State | Input char | Next state / action |
|---|---|---|
| Ground | U+001B | Esc (mark start) |
| Ground | U+0090 DCS, U+0098 SOS, U+009B CSI, U+009D OSC, U+009E PM, U+009F APC | enter matching string/CSI state; also record PC206 |
| Ground | other C1 U+0080–U+009F | PC206 (single char) |
| Esc | `[` | Csi |
| Esc | `]` | OscString |
| Esc | `P`, `X`, `^`, `_` | DcsLikeString(DCS/SOS/PM/APC) |
| Esc | U+001B | emit PC205 for the first ESC; stay in Esc (re-mark) |
| Esc | other final 0x30–0x7E | PC205 ("two-char escape") span = ESC+char |
| Esc | anything else / EOF | PC205 (bare/malformed ESC) |
| Csi | 0x20–0x3F (params/intermediates) | stay (cap applies) |
| Csi | 0x40–0x7E final | PC201, span = full sequence → Ground |
| Csi | anything else / EOF / cap hit | PC205 (unterminated), span = consumed → Ground |
| OscString | BEL (U+0007) or ST (`ESC \` or U+009C) | terminate: parse leading number; `52` → PC203 else PC202; span = full seq → Ground |
| OscString | EOF / cap hit | PC205 unterminated (still classify OSC 52 prefix as PC203) |
| DcsLikeString | ST / EOF / cap | PC204 (or PC205 if unterminated) |

The BEL that terminates an OSC is part of that sequence's span and is not
additionally reported as PC210.

**Layer 2 — residual controls** on chars not consumed by layer 1:
C0 excl. TAB/LF/CR → PC210 (message names BS/BEL specially); DEL → PC211;
TAB → PC212; C1 → PC206.

### 7.3 `invisible` (PC301–PC304)

#### 7.3.1 PC301 codepoint table

U+00AD SOFT HYPHEN; U+034F CGJ; U+115F, U+1160 Hangul choseong/jungseong
filler; U+17B4, U+17B5 Khmer inherent vowels; U+180E Mongolian vowel
separator; U+200B ZWSP; U+200C ZWNJ; U+200D ZWJ (only outside emoji context,
see 7.3.3); U+2060 WORD JOINER; U+2061–U+2064 invisible operators;
U+206A–U+206F deprecated format controls; U+3164 HANGUL FILLER; U+FEFF
ZWNBSP/BOM; U+FFA0 halfwidth Hangul filler; U+FFF9–U+FFFB interlinear
annotation controls; U+1D173–U+1D17A musical format controls;
U+1BCA0–U+1BCA3 shorthand format controls.

Fallback rule: any char with General_Category `Cf` not otherwise claimed by
§7.3.2, §7.3.3, the bidi set (§7.4), or the tag block → PC301. (Keeps us
honest against newly assigned format chars.)

#### 7.3.2 PC302 — tag characters

U+E0000–U+E007F. Always PC302, never context-excused. Message calls out
"hidden ASCII payload (ASCII smuggling)".

#### 7.3.3 Variation selectors and emoji/IVS context (PC303/PC304)

VS set: U+FE00–U+FE0F (VS1–16), U+E0100–U+E01EF (IVS), U+180B–U+180D
(Mongolian FVS).

Context classification (single left-to-right pass, using
Extended_Pictographic from `unicode-properties`):

- **Emoji ZWJ context**: U+200D where the nearest preceding char skipping
  {VS15, VS16, skin tones U+1F3FB–U+1F3FF} is Extended_Pictographic AND the
  nearest following char skipping the same set is Extended_Pictographic
  → PC304 (info). Otherwise ZWJ → PC301.
- **Emoji VS context**: VS15/VS16 immediately after an Extended_Pictographic
  char or after keycap bases `0-9`, `#`, `*` → PC304. Otherwise → PC303.
- **IVS context**: U+E0100–U+E01EF immediately after a Han-script char →
  PC304 (legitimate Japanese kanji variant selection). Otherwise → PC303.
- **Mongolian FVS** after a Mongolian-script char → PC304, else PC303.
- **Run escalation**: ≥ 4 consecutive VS-set chars → the aggregated PC303
  finding for this input is reported at critical (data-smuggling channel).

### 7.4 `bidi` (PC401–PC403)

PC401: U+202A LRE, U+202B RLE, U+202C PDF, U+202D LRO, U+202E RLO.
PC402: U+2066 LRI, U+2067 RLI, U+2068 FSI, U+2069 PDI.
PC403: U+200E LRM, U+200F RLM, U+061C ALM.

Balance annotation: the detector tracks embedding/isolate depth; if the input
ends at nonzero depth, the PC401/PC402 message appends
"(unterminated — affects all following text)". No separate rule code.

### 7.5 `homoglyph` (PC501–PC504) — see ADR-002 for policy rationale

Definitions:

- *Word*: token from `unicode-segmentation::unicode_words()`.
- *skeleton(w)*: TR39 confusable skeleton via `unicode-security`.
- *ASCII ratio*: printable-ASCII chars ÷ all non-whitespace chars of the whole
  input.
- *Script resolution*: per-char `AugmentedScriptSet` (unicode-security);
  a word is *single-script* iff the intersection of its chars' sets is
  non-empty (Common/Inherited chars have the universal set).

#### 7.5.1 PC501 — mixed-script ASCII-skeleton word (critical)

Flag word `w` iff: contains ≥ 1 non-ASCII char, AND contains ≥ 1 ASCII
alphanumeric, AND `w` is NOT single-script, AND `skeleton(w)` is entirely
ASCII. Example hit: `pаypal` (Latin + Cyrillic а). Example non-hit:
`東京tool` (skeleton contains Han → not all-ASCII).

#### 7.5.2 PC502 — whole-script confusable word (warning)

Flag word `w` iff: every char non-ASCII, AND single-script, AND script ∉
{Han, Hiragana, Katakana, Hangul, Bopomofo} (CJK exemption, ADR-002), AND
`skeleton(w)` entirely ASCII, AND input ASCII ratio ≥
`homoglyph.ascii_ratio_threshold` (default 0.80). Example hit: `раураl`
(all Cyrillic) inside an English command. Example non-hit: the same word
inside pasted Russian prose (ratio below threshold).

#### 7.5.3 PC503 — fullwidth forms (warning)

Per-char table: U+FF01–U+FF5E (fullwidth `!`–`~`), U+3000 ideographic space.
Suggestion: "map to ASCII (`sanitize` strategy `ascii_map`)". Consecutive
occurrences merge into one span.

#### 7.5.4 PC504 — lookalike punctuation (warning)

| Codepoints | ASCII target |
|---|---|
| U+2018 U+2019 U+201A U+201B U+00B4 U+02B9 U+02BC U+2032 | `'` |
| U+201C U+201D U+201E U+201F U+2033 | `"` |
| U+2010 U+2011 U+2012 U+2013 U+2014 U+2015 U+2212 | `-` |
| U+2044 U+2215 | `/` |
| U+2024 | `.` |
| U+2026 | `...` |
| U+02DC U+223C | `~` |
| U+02C6 | `^` |
| U+2223 U+2758 | `\|` |

#### 7.5.5 Deduplication and precedence

If every non-ASCII char of a PC501/PC502 candidate word is already reported
under PC503/PC504, suppress the PC501/PC502 finding (the per-char rules carry
an actionable `ascii_map` suggestion; the word-level rule would be noise).
PC503/PC504 always report regardless of word-level findings.

## 8. CLI contract

### 8.1 Command tree

```
pastecheck [check] [FILE|-] [--clipboard] [--format human|json]
           [--fail-on info|warning|critical] [--show-info] [--quiet]
           [--color auto|always|never] [--max-bytes N] [--config PATH]
           [--enable CAT[,CAT…]] [--disable CAT[,CAT…]]
pastecheck sanitize [FILE|-] [--clipboard] [--config PATH] [--max-bytes N]
           [--newline keep|strip-trailing] [--homoglyph keep|ascii-map]
pastecheck gate [--shell zsh|bash] [--config PATH]
pastecheck shell-init <zsh|bash>
pastecheck rules [--format human|json]
```

`check` is the default subcommand: `pastecheck < f`, `pastecheck f.txt`, and
`pbpaste | pastecheck` all work. `CAT` = category name
(`newline|control|invisible|bidi|homoglyph`).

Global env: `PASTECHECK_CONFIG` (config path), `NO_COLOR` (respected),
`PASTECHECK_DISABLE`, `PASTECHECK_GATE_FAIL`, `PASTECHECK_BASH_MODE`
(hook-side, §13).

### 8.2 Exit codes (SEC-14)

| Code | Meaning | Used by |
|---|---|---|
| 0 | success; for `check`/`gate`: clean (§5.2) | all |
| 1 | findings at/above `fail_on` | check, gate |
| 2 | usage error (bad flags, TTY stdin without source) | all |
| 3 | operational error (size cap, IO error, clipboard unavailable, config parse error outside gate) | all |

`sanitize` exits 0 on successful output even when findings existed (it
processed them); 2/3 as above.

### 8.3 `check` behavior

stdin/file/clipboard → analyze → render to stdout (human §9.2 or json §9.3)
→ exit per §8.2. `--quiet` suppresses stdout entirely. Human format goes to
stdout; all diagnostics to stderr.

### 8.4 `sanitize` behavior

Analyze, apply §10 strategies, write sanitized text to **stdout only**
(SEC-6: never in-place, never clipboard). A per-category action summary goes
to **stderr** (§10.4) so stdout stays pipe-clean.

### 8.5 `gate` protocol (hook-facing; stable contract for snippets)

- Input: candidate paste bytes on stdin.
- Output (stdout): empty when clean. When findings ≥ `fail_on`: a **plain
  ASCII-safe, uncolored** compact report, at most `hook.show_max_findings`
  (default 10) finding lines plus a one-line header and an optional
  `… and N more` line. Format:

  ```
  pastecheck: 4 findings (2 critical, 2 warning) in 3 lines
    critical PC401 bidi override U+202E ×3 @ line 1
    critical PC301 invisible U+200B ZERO WIDTH SPACE ×1 @ line 2
    warning  PC504 lookalike punctuation U+2019 ×2 @ line 3
    warning  PC101 internal line feed ×2
  ```

- Exit codes: 0 clean → snippet inserts original; 1 findings → snippet
  prompts; 3 could-not-scan (size cap, config-independent IO failure) →
  snippet shows the stderr message and prompts with paste/abort only (SEC-9);
  2 never occurs with a correct snippet.
- Latency budget: §15.

### 8.6 `shell-init`

Prints the embedded snippet for the named shell to stdout, unmodified, so that
`eval "$(pastecheck shell-init zsh)"` installs the hook. Exit 2 for unknown
shell names. Snippets are versioned with the binary (single artifact — no
skew).

### 8.7 `rules`

Human: the §5.3 table. JSON: array of rule objects
(`code, category, default_severity, name, message, suggestion`) plus
`unicode_data` info (crate versions compiled in via `env!`/`build`-free
constants) so data freshness is observable (research 03 caveat).

## 9. Output rendering

### 9.1 Escaping model (SEC-3, normative)

`render::escape_char(c) -> Escaped` classification, applied to every
input-derived char before display:

1. Printable ASCII 0x20–0x7E → raw.
2. Input LF inside context snippets → rendered as the symbol `␊` (U+240A);
   layout newlines are emitted by the renderer itself (renderer-owned bytes
   are trusted; input-owned bytes are not).
3. Any codepoint in the rule tables of §7 (controls, invisibles, bidi, VS,
   tags) → `<U+XXXX NAME>` where NAME comes from our own tables
   (e.g. `<U+200B ZERO WIDTH SPACE>`); C0 also get caret aliases
   (`<U+001B ESC>`).
4. Other chars: raw iff General_Category ∈ {L*, M*, N*, P*, S*} or is U+0020;
   everything else (other Zs/Zl/Zp, Cc, Cf, Co, Cn, Cs) → `<U+XXXX>`.
   This keeps CJK, emoji, and normal punctuation readable (goal: a Japanese
   command line renders as-is) while making anything suspicious visible.
5. Invalid-UTF8 offenders (PC220) render as `<0xNN>` per byte.

The gate output (§8.5) uses the same function but with rule 4 tightened: any
non-ASCII renders as `<U+XXXX>` (the gate's output area — `zle -M` — is less
predictable, so it stays pure ASCII).

### 9.2 Human format

Header line (`N findings (…) in FILE|stdin|clipboard, X bytes`), then one
block per finding:

```
critical PC301 invisible character  ×3   line 2, col 15
  U+200B ZERO WIDTH SPACE
  context: curl https://examp<U+200B>le.com/install.sh
  hint: `pastecheck sanitize` removes this
```

Context: up to 60 escaped chars centered on the first span, single line per
finding. Color (via anstyle, `--color`/`NO_COLOR`): critical red, warning
yellow, info dim; color never carries information alone. `--show-info`
reveals info findings; default hides them (count still in header as
"+N info"). Clean input prints `pastecheck: clean (X bytes scanned)` to
stdout unless `--quiet`.

### 9.3 JSON format (schema v1, stable)

```json
{
  "schema_version": 1,
  "tool": {"name": "pastecheck", "version": "0.1.0"},
  "source": "stdin|file|clipboard",
  "verdict": "clean|findings",
  "max_severity": "critical",
  "fail_on": "warning",
  "stats": {"bytes": 123, "chars": 120, "lines": 3},
  "findings": [
    {
      "code": "PC301", "category": "invisible", "severity": "critical",
      "count": 3, "spans_truncated": false,
      "spans": [{"start_byte": 28, "end_byte": 31, "line": 1, "column": 20}],
      "offenders": ["U+200B"],
      "message": "invisible character: zero width space",
      "suggestion": "remove"
    }
  ]
}
```

Stability contract: fields may be added in v1.x; never removed/renamed/
retyped without bumping `schema_version`. All strings are JSON-escaped; the
§9.1 guarantee applies to decoded values too (offenders/messages contain
`U+XXXX` notation, never the raw char). Golden-tested (issue 15/23).

## 10. Sanitizer specification (ADR-004)

### 10.1 Strategies (config `[sanitize]`, flags override)

| Category | Default | Alternatives | Action |
|---|---|---|---|
| newline | `keep` | `strip-trailing` | `strip-trailing` removes the final EOL (PC105 span) only |
| control | `strip` | `keep` | remove full sequence spans (PC201–PC205) and single controls (PC206/210/211); TAB (PC212) is always kept; PC220 invalid bytes removed |
| invisible | `strip` | `keep` | remove PC301/302 spans and PC303 occurrences; PC304 (emoji/IVS context) is **always preserved** |
| bidi | `strip` | `strip-all`, `keep` | `strip` removes PC401/402; `strip-all` also removes PC403 marks |
| homoglyph | `keep` | `ascii-map` | `ascii-map` rewrites only PC503/PC504 table chars to their ASCII targets; PC501/PC502 words are **never** auto-rewritten (intent cannot be inferred) — they remain and are re-reported in the summary |

### 10.2 Algorithm

1. Run the full analysis (all detectors, ignoring `fail_on`).
2. Collect removal spans per the strategy table; sort by start; drop spans
   fully contained in an earlier removal (an OSC payload containing a ZWSP is
   removed once).
3. Rebuild the string: copy non-removed regions; apply `ascii-map`
   replacements to remaining chars.
4. Write to stdout (with trailing-newline fidelity — output is exactly the
   rebuilt string, no added EOL).

### 10.3 Guarantees (property-tested, issue 16)

- Idempotent: `sanitize(sanitize(x)) == sanitize(x)`.
- Effective: re-running `check` on sanitized output yields no findings ≥
  warning in categories whose strategy is a strip/map variant (given the same
  config).
- Conservative: no codepoints introduced other than the §7.5.3/§7.5.4 ASCII
  targets; length never grows except via the `…` → `...` mapping.
- PC304 preservation: emoji sequences and Japanese IVS survive byte-exact.

### 10.4 Summary output (stderr)

One line per category with action taken, e.g.
`sanitize: control: removed 2 sequences; homoglyph: kept 1 finding (PC501 — review manually)`.
This satisfies the abuse case "user assumes sanitize fixed everything"
(§3.4).

## 11. Configuration

Path resolution: `--config` flag > `PASTECHECK_CONFIG` >
`$XDG_CONFIG_HOME/pastecheck/config.toml` > `~/.config/pastecheck/config.toml`
(same rule on macOS — CLI convention, not `~/Library`). Missing file = pure
defaults (not an error).

```toml
# all keys optional; shown with defaults
fail_on   = "warning"          # info|warning|critical
max_bytes = 5242880
color     = "auto"             # auto|always|never

[detectors]                    # category master switches
newline = true
control = true
invisible = true
bidi = true
homoglyph = true

[rules]                        # per-rule override: off|info|warning|critical
# "PC212" = "off"
# "PC403" = "info"

[homoglyph]
ascii_ratio_threshold = 0.8    # 0.0–1.0 (§7.5.2)

[allowlist]
codepoints = []                # e.g. ["U+00AD"] — findings whose offenders are
                               # all allowlisted are suppressed

[sanitize]
newline   = "keep"             # keep|strip-trailing
control   = "strip"            # strip|keep
invisible = "strip"            # strip|keep
bidi      = "strip"            # strip|strip-all|keep
homoglyph = "keep"             # keep|ascii-map

[hook]
show_max_findings = 10
```

Precedence: CLI flags > config file > built-in defaults. Validation errors
(bad enum value, threshold out of range, unknown rule code): `check`/
`sanitize` exit 3 with a precise message; `gate` warns on stderr and proceeds
with defaults (SEC-10). Unknown keys: stderr warning, continue. Safety
invariants: `PC220` and `PC302` cannot be set below `warning` (attempting to
→ exit 3 / gate warning); master `control=false` still leaves PC220 active
(it is emitted by input acquisition, §4.4).

## 12. Clipboard adapters (ADR-005)

| Platform condition | Command line |
|---|---|
| macOS | `pbpaste` |
| Linux, `WAYLAND_DISPLAY` set | `wl-paste --no-newline` |
| Linux, else `DISPLAY` set | `xclip -selection clipboard -o` → fallback `xsel --clipboard --output` if xclip absent |
| Neither variable | exit 3: `no clipboard available (need wl-paste or xclip/xsel, or run inside a session with a display)` |

Execution rules (SEC-7): `std::process::Command` with the fixed argv above;
no shell; stdin null; stdout read up to `max_bytes + 1` (over → exit 3 as
§6.2); kill after 5 s; non-zero exit or spawn failure → exit 3 with the
tool's name and a one-line install hint. The captured bytes then enter §6
exactly like stdin bytes.

## 13. Shell integration

Snippets are embedded in the binary (`include_str!`) and printed by
`shell-init` (§8.6). Both snippets: prefix every symbol with `__pastecheck_`
/ `_pastecheck_`; never `eval` content; stdin-pipe transfer only (SEC-8);
respect `PASTECHECK_DISABLE=1` (plain insert, zero calls); locate the binary
once per session (`command -v pastecheck`, overridable via
`PASTECHECK_BIN`).

### 13.1 zsh hook (issue 20; mechanism verified in research 02 §2)

Install: replace the `bracketed-paste` widget with `__pastecheck_paste` via
`zle -N`. If the current widget is neither the builtin nor ours, print a
one-time warning naming the conflicting widget (bracketed-paste-magic /
safe-paste) and replace anyway (documented single-consumer reality).

Widget state machine:

| # | State | Event | Action → next |
|---|---|---|---|
| 1 | CAPTURE | widget invoked | `local PASTED; zle .bracketed-paste PASTED` → 2 (empty paste → INSERT) |
| 2 | GATE | — | `out=$(print -rn -- "$PASTED" \| pastecheck gate --shell zsh 2>>(read stderr))`; rc=$? |
| 3 | rc=0 | — | INSERT original |
| 4 | rc=1 | — | PROMPT: `zle -M` shows gate output + `[p]aste  [a]bort  [s]anitize  [d]etails`; `read -k 1` |
| 5 | rc=3 | — | PROMPT-LIMITED: show stderr message + `[p]aste  [a]bort` |
| 6 | rc=127 or other | — | per `PASTECHECK_GATE_FAIL` (default `open`): INSERT + one-line warning; `closed`: DISCARD + warning |
| 7 | PROMPT key `p` | — | INSERT original |
| 8 | PROMPT key `a` (or Ctrl-C/ESC) | — | DISCARD (`zle -M "pastecheck: paste aborted"`) |
| 9 | PROMPT key `s` | — | `san=$(print -rn -- "$PASTED" \| pastecheck sanitize 2>/dev/null; printf x)`; `san=${san%x}`; INSERT sanitized (spawn failure → back to PROMPT) |
| 10 | PROMPT key `d` | — | re-run gate with `--shell zsh` full output? No — v1: `d` shows up to 20 lines via `zle -M` from `check --format human --color never` and returns to PROMPT |
| 11 | INSERT | — | `LBUFFER+="$content"`; `zle -M ''` clears message |

Programmatic `LBUFFER+=` insertion never triggers `accept-line`, even for
content ending in newline — this is the property that makes `p`/`s` safe.

### 13.2 bash hook (issue 21; research 02 §3)

Primary mode (`PASTECHECK_BASH_MODE=intercept`, default): snippet asserts
bash ≥ 4.4 and `enable-bracketed-paste` support, sets it on, then
`bind -x '"\e[200~": __pastecheck_paste'`. Handler: byte-read loop from
`/dev/tty` under `LC_ALL=C` (`IFS= read -r -s -N1 -t 0.5`) accumulating until
literal `ESC[201~`, hard cap `max_bytes`; then the same gate flow as zsh
rows 2–10 with prompts via `printf >/dev/tty` + `read -r -s -n1 </dev/tty`;
INSERT = splice into `READLINE_LINE`/`READLINE_POINT`. Timeout without
terminator → treat captured bytes as the paste (document; SEC-15 adjacent).
NUL fidelity caveat per research 02 (accepted risk, documented).

Fallback mode (`PASTECHECK_BASH_MODE=keybind`): no paste interception;
`C-x C-v` runs a guarded clipboard insert. The snippet fetches the clipboard
bytes **once** into a shell variable using the same platform tool order as
§12 (`pbpaste` / `wl-paste --no-newline` / `xclip -selection clipboard -o` /
`xsel --clipboard --output`), then runs the standard gate flow on that
variable via stdin (single fetch — no TOCTOU between analyzed and inserted
content), then the p/a/s prompt and READLINE splice. The 5-line duplicated
tool-selection logic is accepted; the binary's §12 adapter remains the
audited path for CLI `--clipboard`. Snippet auto-selects fallback (with
stderr notice) when bash < 4.4.

### 13.3 Common hook requirements

- Gate output is plain ASCII (§8.5/§9.1) — safe for `zle -M` and `/dev/tty`.
- Failure policy per SEC-9; `PASTECHECK_GATE_FAIL` ∈ {open, closed}.
- Trailing-newline-preserving capture (`printf x` sentinel) wherever `$(…)`
  captures content whose trailing bytes matter.
- End-marker truncation (research 02 §1): captured prefix is still gated;
  document the residual in README (SEC-15).
- The snippet never blocks longer than gate latency + user decision; no
  retries, no background jobs.

## 14. Error handling

`thiserror` taxonomy → exit codes: `Usage` → 2; `Io`, `SizeExceeded`,
`ClipboardUnavailable`, `ConfigInvalid` → 3 (with the gate exception for
config, SEC-10). All errors: single line on stderr, prefix `pastecheck: `,
no input content embedded (SEC-2). Panics = bugs; `main` installs no panic
hook (default abort output acceptable; hooks treat it as rc≠{0,1,3} → SEC-9
path).

## 15. Performance requirements

| Path | Budget |
|---|---|
| `gate`, 64 KiB input | ≤ 25 ms scan (excl. spawn), p95 on dev hardware |
| process spawn + gate + exit, typical paste (< 4 KiB) | ≤ 15 ms perceived |
| `check`, 5 MiB input | ≤ 500 ms |
| binary size | ≤ 5 MiB stripped (no heavy data tables beyond unicode crates) |

Enforcement: `#[ignore]`d timing tests with generous CI multipliers, run
locally and in a non-blocking nightly job (issue 23); budgets are review
gates, not hard CI gates (CI variance).

## 16. Testing and validation strategy

Layers (full matrix in ISSUE_PLAN §6):

1. **Unit**: per-detector table-driven tests; §7 tables asserted exactly;
   rule-registry invariants (unique codes, categories match).
2. **Property** (proptest): sanitizer guarantees §10.3; escaping totality
   (any char → output allowlist, SEC-3); engine no-panic on arbitrary strings.
3. **Corpus** (issue 23): `tests/corpus/*.toml`, TOML basic strings with
   `\uXXXX` escapes only — no raw hostile bytes in the repo (reviewable on
   GitHub, which itself warns on bidi). Each case: `id`, `description`,
   `input`, expected `[[case.expect]] {code, count, severity}` +
   optional golden human/JSON files under `tests/golden/`.
   `UPDATE_GOLDEN=1 cargo test` regenerates.
4. **CLI integration** (assert_cmd): exit codes ×
   {clean, findings, usage, size-cap, clipboard-missing}; stdin/file/
   clipboard sources (fake clipboard tools on PATH); config precedence.
5. **Shell E2E** (issue 22): `expect` drives real `zsh -f`/`bash --norc` on a
   pty; sends literal `ESC[200~ … ESC[201~` frames; scenarios: clean insert,
   findings→abort, findings→paste, findings→sanitize, disable env, missing
   binary (fail-open), rc=3 prompt. Runs as a dedicated CI job (ubuntu:
   zsh+bash; macos: zsh).
6. **Fuzz** (issue 24): cargo-fuzz targets `engine`, `ansi`, `sanitize`
   (sanitize target asserts the idempotency/effectiveness properties, not
   just no-panic); seeded from the corpus; scheduled weekly CI job,
   non-blocking but reported.

## 17. Release engineering and supply chain (ADR-006)

- CI (`ci.yml`): jobs `fmt`, `clippy`, `test` (ubuntu-latest, macos-latest),
  `msrv` (cargo check on pinned MSRV), `deny` (cargo-deny), `shell-e2e`.
  Actions SHA-pinned; top-level `permissions: {}`.
- Release (`release.yml`): trigger `push: tags: v*`; matrix

  | Target | Runner |
  |---|---|
  | x86_64-apple-darwin | macos-13 |
  | aarch64-apple-darwin | macos-14 |
  | x86_64-unknown-linux-musl | ubuntu-24.04 (+musl-tools) |
  | aarch64-unknown-linux-musl | ubuntu-24.04-arm (+musl-tools) |

  `cargo build --release --locked`, strip, `pastecheck-<ver>-<target>.tar.gz`
  (binary + LICENSE-* + README), single `SHA256SUMS`, draft GitHub Release.
  Only this workflow's release job has `contents: write`.
- Versioning: SemVer from 0.1.0; `CHANGELOG.md` (Keep a Changelog); tag =
  `v<version>` matching `Cargo.toml`.
- crates.io: manual `cargo publish --locked` by the maintainer per
  `RELEASING.md`; no registry token in CI (research 03 §3).
- main-branch merge ≠ release: releases happen only via tags (owner policy).

## 18. Documentation plan (issue 26)

README (install: release binaries + `cargo install`; quickstart; hook setup
zsh/bash; what-it-catches table with examples; security model summary;
config reference; residual risks per SEC-15), SECURITY.md (SEC-16),
CONTRIBUTING.md (build/test/corpus workflow), RELEASING.md (issue 25),
CHANGELOG.md.

## 19. Traceability

Section → issue coverage lives in `docs/ISSUE_PLAN.md` §5 and is kept in sync
whenever either document changes. GitHub Issues are derived artifacts of
`docs/issues/*.md`; on disagreement, repo docs win and issues get updated.
