# Title

Core data model and rule registry

## Summary

Implement `src/model.rs` (Category, Severity, Span, Offender, Finding, Stats,
Report) and `src/rules.rs` (the static 26-rule registry) exactly as
DESIGN §5, with serde support and registry invariant tests.

## Context

Every detector, renderer, and the config layer consume these types. The rule
registry is the single source of truth for codes, categories, default
severities, names, messages, and sanitize suggestions (DESIGN §5.3).

## Scope

- `src/model.rs`, `src/rules.rs`, unit tests. Serde derives for JSON output
  (consumed later by issue 16).

## Detailed Requirements

1. Types exactly per DESIGN §5.1 (field names and semantics are normative),
   with: `Severity: Ord` (`Info < Warning < Critical`); `Span` byte offsets +
   1-based line/column; `Offender::{Codepoint(u32), RawByte(u8)}` with a
   `Display` impl producing `U+XXXX` (4–6 uppercase hex digits) or `0xNN`.
2. serde: `Category` and `Severity` serialize as lowercase strings;
   `Offender` serializes as its `Display` string.
3. `rules.rs`: `pub struct Rule { pub code: &'static str, pub category:
   Category, pub default_severity: Severity, pub name: &'static str,
   pub message: &'static str, pub suggestion: Option<&'static str> }` and
   `pub static RULES: &[Rule]` containing **all 26 rows of DESIGN §5.3**
   (PC101–PC105, PC201–PC206, PC210–PC212, PC220, PC301–PC304,
   PC401–PC403, PC501–PC504) with messages matching the registry names.
4. Lookup helpers: `rule(code: &str) -> Option<&'static Rule>` and
   `codes() -> impl Iterator`.
5. `Report::max_severity` computed, not stored ad hoc: helper
   `Report::compute(findings, stats)`.

## Acceptance Criteria

- [ ] Registry has exactly 26 entries; codes unique; every code's prefix
      digit matches its category (1=newline, 2=control, 3=invisible, 4=bidi,
      5=homoglyph) — asserted by a unit test.
- [ ] Severity ordering test: `Info < Warning < Critical`.
- [ ] `serde_json::to_string` of a sample `Finding` produces lowercase
      category/severity and `"U+200B"`-style offenders (unit test).
- [ ] `Offender::Codepoint(0x1B).to_string() == "U+001B"`,
      `Codepoint(0x1D173) == "U+1D173"`, `RawByte(0x9B) == "0x9B"`.

## Validation

```sh
cargo test --locked model:: rules::
cargo clippy --all-targets -- -D warnings
```

## Dependencies

Blocked by: 01.

## Non-goals

Detector logic (07–12), codepoint tables (added by detector issues into
`rules.rs` as `const` arrays), JSON schema envelope (16).

## Design References

- `docs/DESIGN.md` §5 (all), §4.4 (ordering consumed by engine)
