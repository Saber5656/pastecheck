# ADR-002: CJK-aware homoglyph policy (no "non-ASCII = suspicious")

Status: accepted (2026-07-08; conservative design decision by the design
agent — flagged to the product owner in the design review PR)

## Context

The primary user (and a large share of the target audience) pastes Japanese
text into terminals daily: CJK ideographs, kana, fullwidth forms, emoji, and
ideographic variation sequences (IVS) are *normal input*, not attacks. A
naive homoglyph detector that alarms on non-ASCII would fire constantly and
get uninstalled — the guard would then protect nobody. At the same time,
confusable substitution (Cyrillic `а` in `pаypal`) and fullwidth accidents
(`ｒｍ`) are real threats/corruptions that must be caught.

## Decision

Homoglyph detection flags only **confusable-with-ASCII** constructs
(DESIGN §7.5):

1. PC501 (critical): mixed-script words whose TR39 skeleton is all-ASCII.
2. PC502 (warning): whole-script confusable words, only when the surrounding
   input is ASCII-majority (`ascii_ratio_threshold`, default 0.80), and never
   for Han/Hiragana/Katakana/Hangul/Bopomofo.
3. PC503/PC504 (warning): explicit fullwidth and lookalike-punctuation tables
   with deterministic ASCII mappings.

Pure CJK text, mixed Japanese/Latin prose (`東京tool`), emoji ZWJ sequences,
and Han+IVS never produce homoglyph or invisible findings by default
(PC304 info records emoji/IVS joiners without alarming).

## Alternatives considered

- **Flag all non-ASCII**: unusable for CJK users; rejected.
- **TR39 restriction levels (identifier profile)**: designed for identifiers,
  not free text; would flag legitimate prose; rejected for v1.
- **User-configured script allowlist as the primary mechanism**: pushes
  correctness onto configuration; kept only as a secondary suppression tool
  (`[allowlist]`).

## Consequences

- False-alarm rate for everyday CJK/emoji input is designed to be ~zero;
  this is validated by dedicated Japanese/Russian/English corpus cases
  (issue 23).
- Whole-word substitutions inside non-ASCII-majority documents (e.g. a
  Cyrillic lookalike word pasted into Russian prose) are intentionally not
  flagged — accepted residual, documented in README.
- The 0.80 threshold and the CJK exemption list are config-visible design
  constants; changing them is a product decision, not an implementation
  detail.
