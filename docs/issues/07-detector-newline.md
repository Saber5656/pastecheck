# Title

Newline detector (PC101–PC105)

## Summary

Implement `src/detectors/newline.rs` producing PC101 (internal LF), PC102
(lone CR), PC103 (internal CRLF), PC104 (line-break lookalikes), PC105
(trailing EOL) exactly per DESIGN §7.1.

## Context

Trailing-EOL-executes-immediately is the canonical paste attack; lone CR both
submits and overwrites display. The trailing/internal split (PC105 vs
PC101–103) drives severity and messaging.

## Scope

- `src/detectors/newline.rs` + its `const` codepoint list for PC104 in
  `rules.rs`; table-driven unit tests.

## Detailed Requirements

1. Trailing EOL first: if text ends with `\r\n` → one PC105 occurrence, span
   covering both bytes, message detail `CRLF`; else if ends with `\n` or
   `\r` → PC105 with detail `LF`/`CR`. The trailing EOL's chars are excluded
   from steps 2–3.
2. Remaining `\r\n` pairs → PC103 (span per pair, chars consumed).
3. Remaining `\n` → PC101; remaining `\r` → PC102.
4. PC104 chars: U+0085, U+000B, U+000C, U+2028, U+2029 (const table).
5. Spans: adjacent same-code occurrences do **not** merge for PC101/102/103
   (each occurrence has meaning: line count) — the FindingsBuilder span cap
   handles volume; PC104 merges adjacent runs per DESIGN §7 shared rules.

## Acceptance Criteria

- [ ] `"ls\n"` ⇒ only PC105 (detail LF), no PC101.
- [ ] `"a\nb\n"` ⇒ PC101 count 1 + PC105.
- [ ] `"a\r\nb\r\n"` ⇒ PC103 count 1 + PC105 (detail CRLF).
- [ ] `"a\rb"` ⇒ PC102 count 1; `"a\r"` ⇒ PC105 (detail CR) only.
- [ ] `"a\u{2028}b\u{0085}c"` ⇒ PC104 count 2.
- [ ] `"single line"` ⇒ no newline findings.
- [ ] Line/column values of spans verified against `line_col`.

## Validation

```sh
cargo test --locked detectors::newline
```

## Dependencies

Blocked by: 05.

## Non-goals

Sanitize strategy (17), any other C0 handling (08/09).

## Design References

- `docs/DESIGN.md` §7.1, §5.3 (PC101–PC105 rows)
- `docs/research/01-terminal-paste-threats.md` §T1
