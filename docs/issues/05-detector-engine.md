# Title

Detector trait, FindingsBuilder, and engine pipeline

## Summary

Implement `src/detectors/mod.rs` (the `Detector` trait), the aggregating
`FindingsBuilder`, and `src/engine.rs` running detectors in fixed order,
per DESIGN §4.4.

## Context

Detectors only report occurrences; all policy (aggregation, caps, severity
overrides, allowlist suppression, ordering) is centralized in
`FindingsBuilder` so it is implemented and tested once (SEC-4).

## Scope

- `src/detectors/mod.rs`, `src/engine.rs`, `EffectiveConfig` minimal struct
  (detector toggles + per-rule severity map + allowlist set + homoglyph
  threshold; full file loading is issue 18), unit tests with a fake detector.

## Detailed Requirements

1. `Detector` trait and `ScanInput` exactly per DESIGN §4.4.
2. `FindingsBuilder::record(code: &'static str, span: Span, offender:
   Option<Offender>, detail: Option<String>)`:
   - Aggregates per rule code: one `Finding` per code per run; `count`
     increments; spans kept ascending, capped at 20 with
     `spans_truncated = true` beyond; offenders deduped, capped at 10.
   - Effective severity = config override if present else registry default;
     override value `off` drops the rule entirely.
   - Allowlist: if **all** offenders of an occurrence are in
     `EffectiveConfig.allowlist_codepoints`, the occurrence is not recorded.
   - `detail` (e.g. "unterminated") is appended to the registry message once.
3. Safety invariant (DESIGN §11): overrides below `warning` for `PC220` and
   `PC302` are rejected at config layer, but the builder also clamps them
   defensively (debug_assert + clamp).
4. `engine::analyze(input: &ScanInput, cfg: &EffectiveConfig) -> Report`:
   runs enabled detectors in fixed order `newline, control, invisible, bidi,
   homoglyph` (control = ANSI parser then residual layer, issues 08/09),
   then sorts findings by (first span start byte, code) and computes stats +
   max severity. Detectors that are disabled by category toggle do not run.
5. Determinism: same input + config ⇒ byte-identical `Report` (unit test
   runs twice and compares Debug output).

## Acceptance Criteria

- [ ] Fake-detector tests prove: aggregation by code, span cap 20 +
      truncation flag, offender cap 10 + dedup, severity override, `off`,
      allowlist suppression, deterministic ordering.
- [ ] Engine skips disabled categories.
- [ ] Empty input ⇒ `Report { findings: [], max_severity: None }`.
- [ ] PC220/PC302 clamp behavior unit-tested.

## Validation

```sh
cargo test --locked engine:: detectors::
```

## Dependencies

Blocked by: 04.

## Non-goals

Real detectors (07–12), config file parsing (18), PC220 emission (06).

## Design References

- `docs/DESIGN.md` §4.4, §5.2, §3.3 SEC-4, §11 (safety invariants)
