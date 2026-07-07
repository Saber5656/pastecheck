# Title

Escaped attack corpus and golden output tests

## Summary

Create the product-level test corpus of DESIGN §16.3: TOML case files with
`\u`-escaped hostile inputs, expected findings per case, golden human/JSON
outputs, the SEC-3 whole-report output-safety assertion, and the `#[ignore]`
timing tests for §15 budgets.

## Context

The corpus is the product's regression gate: every threat class from
research 01 gets at least one realistic case, stored fully escaped so the
repository never contains raw hostile bytes (reviewable on GitHub, which
itself warns on bidi chars).

## Scope

- `tests/corpus/*.toml`, `tests/corpus.rs` runner, `tests/golden/{human,json}/`,
  corpus `README.md`, timing tests, nightly non-blocking perf job note in CI.

## Detailed Requirements

1. Corpus format (normative):

   ```toml
   [[case]]
   id = "bidi-001-trojan-source"
   description = "CVE-2021-42574 comment-reorder sample"
   input = "if (x != \"user\u202E \u2066// admin\u2069 \u2066\") {"
   golden = true            # human+json golden files exist for this id
   [[case.expect]]
   code = "PC401"
   count = 1
   severity = "critical"
   [[case.expect]]
   code = "PC402"
   count = 3
   ```

   Runner asserts: exact finding set (no unexpected codes ≥ warning),
   counts, severities; golden cases additionally compare `--color never`
   human and JSON output byte-exact; `UPDATE_GOLDEN=1` regenerates.
2. Required cases (minimum 30; at least): trailing-LF one-liner; lone CR;
   CRLF script; U+2028 lookalike; CSI color; OSC title; **OSC 52**; DCS;
   unterminated OSC; C1 CSI (U+009B); BS-overwrite `evil\b\b\b\bgood`;
   NUL; invalid UTF-8 (PC220, non-golden JSON offsets note case); ZWSP in
   URL; BOM prefix; tag-character smuggle; VS run ≥ 4; family-emoji
   negative; keycap negative; Han+IVS negative; Trojan Source sample;
   balanced-isolates; RLM-only prose (warning); `pаypal` PC501; whole-word
   Cyrillic PC502 positive + Russian-prose negative; fullwidth `ｒｍ`;
   smart-quote/em-dash command; mixed JA/EN clean negative
   (`東京で ls -la を実行`); ASCII-only clean; multi-category combined
   sample (golden showcase).
3. Output-safety assertion (SEC-3): for **every** corpus case, run the human
   renderer (`--color never` and `--color always`) and gate formatter, and
   assert the emitted bytes satisfy the §9.1 allowlist (no raw C0 except
   `\n`, no C1/ESC/bidi/invisible/tag codepoints). This is the product-level
   proof that reports are inert.
4. Timing tests (`#[ignore]`): §15 budgets — 64 KiB gate scan ≤ 25 ms
   (library), 5 MiB check ≤ 500 ms — with a 10× CI multiplier env var;
   nightly workflow runs them non-blocking (`continue-on-error: true`,
   results in job summary).
5. Corpus `README.md`: how to add a case, escaping rules (`\uXXXX` only,
   never raw), regeneration workflow.

## Acceptance Criteria

- [ ] ≥ 30 cases covering every rule code PC101–PC504 at least once
      (coverage asserted by the runner: every registry code appears in ≥ 1
      case's expectations — PC212/PC304 via info-level cases).
- [ ] Negative cases (emoji/IVS/CJK/Russian-prose) produce zero findings ≥
      warning — asserted.
- [ ] Golden files byte-stable across two consecutive runs.
- [ ] SEC-3 assertion passes for all cases and both color modes.
- [ ] `grep -rP '[\x00-\x08\x0B-\x1F\x7F]|\xE2\x80[\xAA-\xAE]' tests/corpus/`
      finds nothing (no raw hostile bytes committed).

## Validation

```sh
cargo test --locked --test corpus
UPDATE_GOLDEN=1 cargo test --locked --test corpus && git diff --exit-code
cargo test --locked --test corpus -- --ignored   # timing, local only
```

## Dependencies

Blocked by: 07, 08, 09, 10, 11, 12, 15, 16.

## Non-goals

Fuzzing (25), shell E2E (23), benchmark suite (criterion — not planned).

## Design References

- `docs/DESIGN.md` §16.3, §9.1, §15, §3.3 SEC-3, SEC-4
- `docs/research/01-terminal-paste-threats.md` (case sources)
