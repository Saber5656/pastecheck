# Title

Homoglyph detector (PC501–PC504) with CJK-aware policy

## Summary

Implement `src/detectors/homoglyph.rs`: mixed-script ASCII-skeleton words
(PC501), whole-script confusable words under the ASCII-ratio condition
(PC502), fullwidth forms (PC503), lookalike punctuation (PC504), and the
§7.5.5 dedup rule — exactly per DESIGN §7.5 and ADR-002.

## Context

This detector must catch `pаypal`-class substitution while producing zero
findings for everyday Japanese text. The algorithms and their exemptions are
normative product policy (ADR-002), not tuning targets.

## Scope

- `src/detectors/homoglyph.rs`; PC503/PC504 const tables in `rules.rs`;
  dependencies `unicode-security`, `unicode-script`, `unicode-segmentation`
  (allowed by ADR-001).

## Detailed Requirements

1. Tokenize with `unicode_words()`; compute the input's ASCII ratio once
   (printable-ASCII chars ÷ non-whitespace chars).
2. Per word implement PC501 and PC502 conditions **exactly** as DESIGN
   §7.5.1/§7.5.2, using `AugmentedScriptSet` intersection for single-script
   and `unicode-security` skeleton; CJK exemption set {Han, Hiragana,
   Katakana, Hangul, Bopomofo}; threshold from
   `EffectiveConfig.homoglyph_ascii_ratio_threshold` (default 0.80).
3. PC503 table: U+FF01–U+FF5E, U+3000; PC504 table exactly per §7.5.4 (with
   ASCII target stored per row — consumed by sanitize `ascii-map`).
4. Dedup (§7.5.5): suppress a PC501/PC502 word finding when every non-ASCII
   char of the word is already covered by PC503/PC504 occurrences.
5. Word spans: byte range of the word (PC501/502); char runs merge
   (PC503/504).
6. Offender lists: for word findings, the non-ASCII chars (≤ 10 distinct);
   message includes the escaped skeleton, e.g.
   `looks like "paypal"` — escaped via issue 14's function when rendered
   (store the ASCII skeleton string in detail; it is ASCII by construction).

## Acceptance Criteria

- [ ] `"p\u{0430}ypal"` (Cyrillic а) ⇒ PC501, message contains
      `looks like "paypal"`.
- [ ] PC502 positive/negative pair: an **all-Cyrillic** word whose TR39
      skeleton is all-ASCII — primary candidate `ѕср` (U+0455 U+0441 U+0440,
      skeleton `scp`; verify against the crate's data at implementation
      time and substitute another all-Cyrillic ASCII-skeleton word if
      needed) — inside an English command line ⇒ PC502; the same word
      embedded in Russian prose (ASCII ratio < 0.8) ⇒ no PC502.
- [ ] `"東京tool"` ⇒ no homoglyph findings (skeleton non-ASCII ⇒ PC501 no;
      mixed ⇒ PC502 no).
- [ ] `"使う"` / pure-kana / pure-Han words ⇒ never flagged (CJK exemption).
- [ ] `"ｒｍ −ｒｆ"` ⇒ PC503 (fullwidth) + PC504 (U+2212), and **no**
      PC501/PC502 for those words (dedup rule).
- [ ] `"--flag"` with U+2014 em dash ⇒ PC504 with target `-`.
- [ ] Smart quotes `“cmd”` ⇒ PC504 count 2 target `"`.
- [ ] ASCII-only input ⇒ zero homoglyph findings.
- [ ] Threshold respected: same input, threshold 0.99 vs 0.5 flips a PC502
      case (table-driven).

## Validation

```sh
cargo test --locked detectors::homoglyph
```

## Dependencies

Blocked by: 05.

## Non-goals

Auto-rewriting words (never — ADR-004), URL/punycode analysis (v2),
restriction-level profiles (rejected, ADR-002).

## Design References

- `docs/DESIGN.md` §7.5 (all subsections normative)
- `docs/decisions/ADR-002-cjk-aware-homoglyph-policy.md`
- `docs/research/03-rust-crates-and-release.md` §1 (crate APIs)
