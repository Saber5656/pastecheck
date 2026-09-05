# Title

Bidi control detector (PC401–PC403)

## Summary

Implement `src/detectors/bidi.rs`: explicit overrides/embeddings (PC401),
isolates (PC402), directional marks (PC403), with unterminated-depth
annotation, per DESIGN §7.4.

## Context

Trojan Source (CVE-2021-42574): bidi controls reorder displayed text without
changing execution order — review-proof malicious pastes. Marks (LRM/RLM/ALM)
are common in legitimate RTL text, hence warning not critical.

## Scope

- `src/detectors/bidi.rs`; const tables in `rules.rs`.

## Detailed Requirements

1. PC401: U+202A, U+202B, U+202C, U+202D, U+202E. PC402: U+2066–U+2069.
   PC403: U+200E, U+200F, U+061C.
2. Depth tracking: +1 on LRE/RLE/LRO/RLO, −1 on PDF (floor 0); +1 on
   LRI/RLI/FSI, −1 on PDI (floor 0); if either depth > 0 at end of input,
   append detail `(unterminated — affects all following text)` to the
   respective finding's message (once).
3. Offenders named: e.g. `U+202E RIGHT-TO-LEFT OVERRIDE`.
4. No merging across different codes; adjacent same-code chars merge spans.

## Acceptance Criteria

- [ ] The Trojan Source comment sample
      `"… \u{202E} \u{2066}// admin\u{2069} \u{2066}"` ⇒ PC401 count 1 +
      PC402 count 3, both with unterminated detail.
- [ ] Balanced `"\u{2066}abc\u{2069}"` ⇒ PC402 count 2, no unterminated
      detail.
- [ ] `"\u{200F}שלום"` ⇒ PC403 only (severity warning).
- [ ] PDF/PDI without opener does not underflow (floor 0, no panic).
- [ ] Arabic prose without controls ⇒ zero bidi findings (RTL text alone is
      not a finding).

## Validation

```sh
cargo test --locked detectors::bidi
```

## Dependencies

Blocked by: 05.

## Non-goals

Visual reordering simulation; homoglyphs (12).

## Design References

- `docs/DESIGN.md` §7.4, §5.3 (PC401–PC403 rows)
- `docs/research/01-terminal-paste-threats.md` §T4
