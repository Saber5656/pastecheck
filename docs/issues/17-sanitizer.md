# Title

Sanitizer engine and `sanitize` subcommand

## Summary

Implement `src/sanitize.rs` and the `sanitize` subcommand per DESIGN §10 and
ADR-004: per-category strategies, span-based removal with containment dedup,
`ascii-map` for PC503/PC504 only, PC304 preservation, stderr action summary,
and the §10.3 property guarantees.

## Context

Sanitize is the remedy path; its guarantees (idempotent, effective,
conservative, emoji/IVS-preserving) are what make the hook's `s` key safe to
press without reading a diff.

## Scope

- `src/sanitize.rs`; `sanitize` subcommand wiring (flags `--newline`,
  `--homoglyph`, `--clipboard`, FILE/stdin); property tests (proptest).

## Detailed Requirements

1. Strategy resolution: CLI flags > config `[sanitize]` > defaults
   (newline keep; control strip; invisible strip; bidi strip;
   homoglyph keep) — table in DESIGN §10.1 is normative, including:
   TAB always kept; PC304 occurrences never removed; PC403 marks kept under
   `strip`, removed under `strip-all`; PC501/PC502 never rewritten.
2. Algorithm per §10.2: full analysis (ignore `fail_on`); build removal set
   from strategies; sort spans by start; drop spans fully contained in an
   earlier removal; rebuild string; apply `ascii-map` replacements
   (PC503 → ASCII via fullwidth offset / U+3000 → space; PC504 → table
   target, U+2026 → `...`) to surviving chars only.
3. Output: rebuilt string to stdout exactly (no added trailing newline —
   byte-fidelity except removals/replacements).
4. stderr summary per §10.4: one line per category that had findings, action
   counts, and explicit `kept N finding(s) (…) — review manually` for kept
   categories. `--quiet` is not a sanitize flag (summary always printed;
   redirecting stderr is the opt-out).
5. Exit codes: 0 on success (even with kept findings), 2/3 per §8.2.

## Acceptance Criteria

- [ ] `printf 'a\x1b[31mb' | pastecheck sanitize` ⇒ stdout `ab`, summary
      names control: removed 1 sequence.
- [ ] OSC containing ZWSP removed once (containment dedup test).
- [ ] `printf 'l\u200Bs\n' | pastecheck sanitize` ⇒ `ls\n` (newline kept).
- [ ] `--newline strip-trailing` removes only the final EOL.
- [ ] Family emoji and `邉\u{E0100}` survive byte-exact (PC304).
- [ ] `--homoglyph ascii-map`: `ｒｍ —rf “x”` ⇒ `rm -rf "x"`; default keeps
      and summary reports kept PC503/PC504.
- [ ] PC501 word never rewritten under any flag; summary flags it.
- [ ] PC220 invalid bytes removed under control strip.
- [ ] Properties (proptest, 10k cases): idempotent; effective (§10.3 —
      re-check of output has no ≥ warning findings in strip/map categories);
      conservative (no new codepoints beyond documented ASCII targets;
      length non-increasing except `…`→`...`).

## Validation

```sh
cargo test --locked sanitize::
cargo test --locked --test '*' -- sanitize_props
```

## Dependencies

Blocked by: 05, 07, 08, 09, 10, 11, 12.

## Non-goals

Clipboard write-back (never in v1 — SEC-6), gate integration (20/21/22),
in-place file editing.

## Design References

- `docs/DESIGN.md` §10 (normative), §8.4
- `docs/decisions/ADR-004-sanitize-semantics.md`
