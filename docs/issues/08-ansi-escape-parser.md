# Title

ANSI escape parser detector (PC201–PC205)

## Summary

Implement the bounded ECMA-48 state machine of DESIGN §7.2 layer 1 in
`src/detectors/control.rs`: CSI (PC201), OSC (PC202), OSC 52 (PC203),
DCS/SOS/PM/APC (PC204), bare/malformed/unterminated ESC (PC205), including
C1-initiated sequences.

## Context

Escape sequences are the terminal-injection class (research 01 §T2). The
parser must be explicit-state, regex-free, and hard-capped per sequence
(SEC-4) — it is also a fuzz target (issue 25).

## Scope

- Layer-1 parser in `src/detectors/control.rs` with a
  `consumed: Vec<Range<usize>>` output the residual layer (issue 09) uses.
  The transition table of DESIGN §7.2 is normative.

## Detailed Requirements

1. States: `Ground`, `Esc`, `Csi`, `OscString`, `DcsLikeString(kind)`;
   per-sequence cap 4096 chars — cap hit ⇒ PC205 (unterminated) with span =
   consumed prefix, return to Ground.
2. Entry points: U+001B from Ground; C1 openers U+0090/U+0098/U+009B/
   U+009D/U+009E/U+009F from Ground (these also record PC206 via the
   residual layer's shared table — emit the occurrence here with code PC206
   *and* continue parsing the sequence).
3. CSI: params/intermediates 0x20–0x3F loop; final 0x40–0x7E ⇒ PC201, span =
   whole sequence. Any other char ⇒ PC205 (span = consumed), reprocess that
   char in Ground.
4. OSC: terminated by BEL or ST (`ESC \` or U+009C); leading decimal number
   parsed; `52` ⇒ PC203 else PC202. The terminating BEL/ST is inside the
   span and not separately reported. Unterminated at EOF ⇒ PC205 but keep
   the PC203 classification detail if the prefix was `52;`.
5. DCS/SOS/PM/APC: same string-consumption rule ⇒ PC204 / PC205 when
   unterminated.
6. `ESC ESC` ⇒ PC205 for the first ESC, re-mark on the second.
   Two-char escapes (`ESC` + final 0x30–0x7E other than `[ ] P X ^ _`) ⇒
   PC205 with detail "two-char escape", span = both chars.
7. Every consumed range is recorded so issue 09 skips those chars.

## Acceptance Criteria

- [ ] `"\x1b[31m"` ⇒ PC201 count 1, span 0..5.
- [ ] `"\x1b]0;title\x07"` ⇒ PC202; `"\x1b]52;c;aGk=\x07"` ⇒ PC203 (no PC202).
- [ ] `"\x1b]52;c;aGk="` (no terminator) ⇒ PC205 with OSC-52 detail.
- [ ] `"\x1bP…\x1b\\"` ⇒ PC204; `"\u{009B}31m"` ⇒ PC206 + PC201.
- [ ] `"\x1b"` alone ⇒ PC205; `"\x1b\x1b[m"` ⇒ PC205 + PC201.
- [ ] 5000-char OSC payload ⇒ PC205 via cap; parser returns to Ground and
      continues (subsequent `"\x1b[m"` still detected).
- [ ] BEL inside an OSC span is not reported as PC210 (integration test with
      issue 09 once merged; here: `consumed` covers the BEL).
- [ ] Parser never panics on any `&str` (proptest: arbitrary strings).

## Validation

```sh
cargo test --locked detectors::control::ansi
cargo test --locked --test '*' -- ansi_proptest
```

## Dependencies

Blocked by: 05.

## Non-goals

Residual C0/C1/DEL/TAB scanning (09), fuzz harness (25).

## Design References

- `docs/DESIGN.md` §7.2 (transition table is normative), §3.3 SEC-4
- `docs/research/01-terminal-paste-threats.md` §T2
