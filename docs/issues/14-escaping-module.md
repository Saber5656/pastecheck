# Title

Output escaping module (SEC-3)

## Summary

Implement `render::escape` per DESIGN §9.1: the classification function that
makes every input-derived character safe to print, with the strict-ASCII
variant used by `gate`, and property tests proving the output allowlist.

## Context

The report itself is an injection surface (trust boundary B5): a scanner that
re-emits raw hostile bytes weaponizes its own output. This module is the
single choke point every renderer must use; SEC-3 is verified here and again
product-wide in issue 24.

## Scope

- `src/render/mod.rs`: `escape_char`, `escape_str`, `escape_str_ascii`
  (strict variant), codepoint-name lookup backed by the `rules.rs` tables,
  `<0xNN>` byte form for PC220 offenders.

## Detailed Requirements

1. Classification exactly per §9.1 rules 1–5:
   - printable ASCII raw;
   - renderer-owned layout newlines are emitted by callers, never by
     escaping input LF (input LF renders as `␊` U+240A in context snippets);
   - all §7 table codepoints ⇒ `<U+XXXX NAME>` (names from our tables; C0
     get aliases like `ESC`, `BEL`, `BS`, `NUL`, `TAB`);
   - otherwise raw iff General_Category ∈ {L*, M*, N*, P*, S*} or U+0020;
     else `<U+XXXX>`;
   - hex uppercase, 4 digits minimum, 6 for astral (`U+1D173`).
2. Strict variant (`escape_str_ascii`): identical except **any** non-ASCII
   char ⇒ `<U+XXXX>` (used by gate §8.5).
3. Output allowlist property (SEC-3): for arbitrary input strings, the
   escaped output contains only: printable ASCII, `␊`, and (non-strict
   variant) chars of GC classes {L*, M*, N*, P*, S*} that are not in any §7
   table. proptest over arbitrary `String` + explicit corpus of every table
   codepoint.
4. No allocation churn pathologies: `escape_str` over a 5 MiB hostile string
   completes in linear time (debug-only timing sanity test, `#[ignore]`).
5. Table-name completeness is allowed to grow with detector issues: unknown
   table members render `<U+XXXX>` without a name — never panic, never fall
   through to raw.

## Acceptance Criteria

- [ ] `escape_str("a\u{202E}b")` == `a<U+202E RIGHT-TO-LEFT OVERRIDE>b`
      (name present once issue 11's table exists; `<U+202E>` before that —
      test written against the function's contract, updated by 11).
- [ ] `escape_str("日本語ok🎉")` == itself (CJK/emoji raw in non-strict).
- [ ] `escape_str_ascii("日本語")` == `<U+65E5><U+672C><U+8A9E>`.
- [ ] `escape_str("\x1b[31m")` == `<U+001B ESC>[31m` (only ESC is a control;
      `[31m` is printable ASCII).
- [ ] proptest allowlist property passes 10k cases.
- [ ] Every codepoint in every §7 const table maps to non-raw output
      (exhaustive loop test over the tables).

## Validation

```sh
cargo test --locked render::escape
```

## Dependencies

Blocked by: 04. (Names enrich as 07–12 land; contract independent.)

## Non-goals

Layout/coloring (15), JSON encoding (16 — serde handles JSON string
escaping; values still use these functions for offender/message content).

## Design References

- `docs/DESIGN.md` §9.1 (normative), §3.3 SEC-3
- `docs/research/01-terminal-paste-threats.md` §T2 (why output is a boundary)
