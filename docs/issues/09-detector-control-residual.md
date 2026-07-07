# Title

Residual control-character detector (PC206, PC210–PC212)

## Summary

Implement layer 2 of the control detector (DESIGN §7.2): scan characters
**not consumed** by the ANSI parser for C1 (PC206), other C0 (PC210), DEL
(PC211), and TAB (PC212).

## Context

Controls outside escape sequences still manipulate display (BS overwrite,
BEL) or paste behavior (TAB completion in unbracketed pastes). This layer
completes the `control` category together with issue 08.

## Scope

- Residual scan in `src/detectors/control.rs`, consuming the `consumed`
  ranges from the layer-1 parser.

## Detailed Requirements

1. Skip every char inside a consumed range (ranges are sorted,
   non-overlapping; walk with a two-pointer scan — O(n)).
2. Classification of remaining chars:
   - U+0080–U+009F ⇒ PC206 (openers already handled in layer 1 are inside
     consumed ranges and never reach here).
   - C0 excluding TAB/LF/CR ⇒ PC210. Message detail names BS (U+0008,
     "display overwrite"), BEL (U+0007), NUL (U+0000) specially; other C0
     generic.
   - U+007F ⇒ PC211.
   - U+0009 ⇒ PC212.
3. LF/CR are owned by the newline detector — never reported here.
4. Adjacent same-code runs merge into one span (shared rule §7).

## Acceptance Criteria

- [ ] `"a\x08b\x07c"` ⇒ PC210 count 2 with BS and BEL offenders.
- [ ] `"a\x00b"` ⇒ PC210 with NUL offender (no panic).
- [ ] `"a\x7fb"` ⇒ PC211; `"a\tb"` ⇒ PC212 (severity info).
- [ ] `"\u{0085}"` ⇒ **no** PC206 (NEL belongs to PC104/newline) — explicit
      test pinning the category split. NEL is U+0085 which is C1: the
      newline detector claims it; residual scan must exclude U+0085 from
      PC206.
- [ ] `"\x1b[31m\x07"` ⇒ PC201 + PC210(BEL): the BEL after the CSI final is
      residual; the BEL terminating an OSC (`"\x1b]0;t\x07"`) is not.
- [ ] Run merging: `"\x00\x00\x00"` ⇒ PC210 count 3, one span 0..3.

## Validation

```sh
cargo test --locked detectors::control::residual
```

## Dependencies

Blocked by: 05, 08.

## Non-goals

Invalid UTF-8 (06/PC220), escape sequences (08).

## Design References

- `docs/DESIGN.md` §7.2 layer 2, §5.3 (PC206/PC210–PC212 rows), §7.1 (U+0085
  ownership)
