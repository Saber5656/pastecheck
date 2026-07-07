# Title

JSON output (schema v1) for `check` and `rules`

## Summary

Implement `src/render/json.rs` per DESIGN §9.3: the stable schema-v1
envelope for `check --format json`, and `rules --format json`, with golden
tests pinning the schema.

## Context

JSON is the machine contract for scripts and future integrations; stability
rules (add-only within v1) start at first release, so the shape must be
exactly §9.3 from the start.

## Scope

- `src/render/json.rs`; serde serializers on model types (from 04);
  `rules --format json` array per §8.7.

## Detailed Requirements

1. Envelope fields exactly per §9.3: `schema_version` (integer 1), `tool`
   {name, version}, `source`, `verdict`, `max_severity` (string or null),
   `fail_on`, `stats` {bytes, chars, lines}, `findings` array. When PC220
   present: `"note"` field with the §6.3 offsets caveat.
2. Finding objects: `code`, `category`, `severity`, `count`,
   `spans_truncated`, `spans` (array of {start_byte, end_byte, line,
   column}), `offenders` (strings `U+XXXX`/`0xNN`), `message`, `suggestion`
   (string or null). Info findings always included (no `--show-info` effect).
3. Output is a single line (no pretty print) terminated by one `\n`;
   `--quiet` still suppresses stdout entirely.
4. All strings JSON-escaped by serde; message/offender values are already
   `U+XXXX` notation (never raw offending chars — SEC-3 applies to decoded
   values; assert in tests by decoding and running the allowlist check).
5. `rules --format json`: array of registry rows (code, category,
   default_severity, name, message, suggestion) plus `unicode_data` object
   with compiled-in crate versions (§8.7).

## Acceptance Criteria

- [ ] Golden JSON for the multi-category corpus sample matches byte-exact
      (stable field order via struct definition order).
- [ ] `jq` round-trip: `pastecheck --format json < sample | jq -e
      '.schema_version == 1 and (.findings | length) > 0'`.
- [ ] Decoded string values pass the SEC-3 allowlist property.
- [ ] `printf 'ok' | pastecheck --format json` ⇒ `verdict:"clean"`,
      `max_severity:null`, exit 0.
- [ ] `rules --format json | jq 'length'` == 26.

## Validation

```sh
cargo test --locked render::json
printf 'a\xe2\x80\xaeb' | cargo run -q -- --format json | jq .   # a + U+202E + b
```

## Dependencies

Blocked by: 04, 13.

## Non-goals

Schema v2 features, SARIF or other formats (not planned), gate output (20).

## Design References

- `docs/DESIGN.md` §9.3 (normative schema), §8.7, §6.3
