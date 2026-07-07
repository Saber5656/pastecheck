# Title

Human report renderer

## Summary

Implement `src/render/human.rs` per DESIGN §9.2: header, per-finding blocks
with escaped context snippets, severity coloring with `--color`/`NO_COLOR`,
`--show-info`, and the clean-input line — replacing the issue-13 placeholder.

## Context

This is the primary UX for `check`. Everything input-derived flows through
issue 14's escaping (SEC-3); layout bytes (newlines, indentation, color
codes) are renderer-owned and trusted.

## Scope

- `src/render/human.rs`; wiring in `check`; golden tests under
  `tests/golden/human/`.

## Detailed Requirements

1. Header: `N findings (X critical, Y warning[, +Z info]) in <source>,
   <bytes> bytes` — source ∈ {stdin, <path>, clipboard}.
2. Per finding (ordered as engine emits): line 1
   `<severity> <code> <name>  ×<count>   line L, col C` (first span);
   line 2 offenders (`U+XXXX NAME`, comma-joined, `…` if capped);
   line 3 `context: <escaped snippet>` — up to 60 escaped chars centered on
   the first span, single line (input LF as `␊`); line 4
   `hint: <suggestion>` when the registry has one. `spans_truncated` adds
   `(+more locations)` to line 1.
3. Colors via anstyle: critical red bold, warning yellow, info dim;
   `--color auto` = TTY detection (std `IsTerminal`), `NO_COLOR` respected;
   color never the only signal (severity word always printed).
4. `--show-info` reveals info findings; default hides them but counts them
   in the header (`+Z info`).
5. Clean: `pastecheck: clean (<bytes> bytes scanned)` unless `--quiet`.
6. Output size bound (SEC-4): at most `min(findings, 100)` blocks; beyond ⇒
   final line `… and N more findings (use --format json for all)`.
7. PC220 note (§6.3): when present, one trailing line
   `note: input contained invalid UTF-8; offsets refer to the decoded text`.

## Acceptance Criteria

- [ ] Golden tests (fixed `--color never`) for: multi-category input,
      clean input, info-hidden vs `--show-info`, truncated spans, PC220
      note, >100-findings cap.
- [ ] Property: rendered output of hostile corpus passes the SEC-3 output
      allowlist check (reuses issue-14 property harness on whole reports).
- [ ] `--color always` emits SGR; `--color never` and `NO_COLOR=1` emit none
      (byte-exact goldens).
- [ ] Exit codes unchanged by rendering (renderer returns `io::Result`, no
      policy).

## Validation

```sh
cargo test --locked render::human
UPDATE_GOLDEN=1 cargo test --locked render::human   # regeneration path works
git diff --exit-code tests/golden
```

## Dependencies

Blocked by: 13, 14.

## Non-goals

JSON (16), gate compact format (20), localization (v2).

## Design References

- `docs/DESIGN.md` §9.2, §9.1, §3.3 SEC-3, SEC-4
