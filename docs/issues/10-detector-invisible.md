# Title

Invisible-character detector with emoji/IVS context (PC301–PC304)

## Summary

Implement `src/detectors/invisible.rs`: the PC301 invisible/format table with
Cf fallback, PC302 tag characters, and the emoji/IVS context classification
that routes ZWJ and variation selectors to PC304 (legitimate) or PC301/PC303
(suspicious), per DESIGN §7.3.

## Context

This detector carries the highest false-positive risk for the primary
audience (emoji and Japanese IVS text are everyday input). The context rules
of §7.3.3 are what make the tool usable; they are normative, not heuristics
to simplify.

## Scope

- `src/detectors/invisible.rs`; `const` tables in `rules.rs`
  (PC301 explicit list, VS set); uses `unicode-properties` for
  General_Category and Extended_Pictographic (add dependency — allowed by
  ADR-001).

## Detailed Requirements

1. PC301 explicit table exactly per DESIGN §7.3.1 (19 entries/ranges:
   U+00AD, U+034F, U+115F, U+1160, U+17B4, U+17B5, U+180E, U+200B, U+200C,
   U+200D*, U+2060, U+2061–U+2064, U+206A–U+206F, U+3164, U+FEFF, U+FFA0,
   U+FFF9–U+FFFB, U+1D173–U+1D17A, U+1BCA0–U+1BCA3). `*` ZWJ subject to
   context (req 3).
2. Cf fallback: any `General_Category == Format` char not in: the explicit
   table, the VS set, bidi set (U+200E, U+200F, U+061C, U+202A–U+202E,
   U+2066–U+2069), or tag block ⇒ PC301.
3. Context classification (single pass, per DESIGN §7.3.3):
   - ZWJ with Extended_Pictographic on both sides (skipping VS15/VS16 and
     skin tones U+1F3FB–U+1F3FF when looking) ⇒ PC304; else PC301.
   - VS15/VS16 after Extended_Pictographic or keycap base (`0`–`9`, `#`,
     `*`) ⇒ PC304; else PC303.
   - U+E0100–U+E01EF after a Han-script char (use `unicode-script`) ⇒ PC304;
     else PC303.
   - U+180B–U+180D after a Mongolian-script char ⇒ PC304; else PC303.
   - ≥ 4 consecutive VS-set chars ⇒ the run's occurrences report at
     escalated severity critical (detail "variation-selector run —
     possible data smuggling"); escalation applies to the aggregated PC303
     finding.
4. PC302: U+E0000–U+E007F, always, no context excuse.
5. Adjacent same-code occurrences merge spans (shared rule).

## Acceptance Criteria

- [ ] `"curl https://examp\u{200B}le.com"` ⇒ PC301 (ZWSP).
- [ ] `"\u{FEFF}ls"` ⇒ PC301 (BOM offender named ZERO WIDTH NO-BREAK SPACE).
- [ ] Family emoji `"👨\u{200D}👩\u{200D}👧"` ⇒ PC304 only, **no PC301**.
- [ ] Keycap `"1\u{FE0F}\u{20E3}"` ⇒ PC304 only.
- [ ] `"邉\u{E0100}"` (Han + IVS) ⇒ PC304 only; `"a\u{E0100}"` ⇒ PC303.
- [ ] `"a\u{200D}b"` (ZWJ between letters) ⇒ PC301.
- [ ] `"x\u{FE00}\u{FE01}\u{FE02}\u{FE03}"` ⇒ PC303 at severity critical
      (run escalation).
- [ ] Tag smuggle `"hi\u{E0041}\u{E0042}"` ⇒ PC302 count 2.
- [ ] Plain Japanese `"東京でツールを使う"` ⇒ zero invisible findings.
- [ ] Cf fallback catches a Cf char absent from every explicit table:
      `"a\u{110BD}b"` (U+110BD KAITHI NUMBER SIGN, GC=Cf) ⇒ PC301.

## Validation

```sh
cargo test --locked detectors::invisible
```

## Dependencies

Blocked by: 05.

## Non-goals

Bidi chars (11), sanitize preservation of PC304 (17), NBSP/visible
whitespace (v2).

## Design References

- `docs/DESIGN.md` §7.3 (tables and context rules are normative)
- `docs/research/01-terminal-paste-threats.md` §T3
- `docs/decisions/ADR-002-cjk-aware-homoglyph-policy.md` (CJK/emoji stance)
