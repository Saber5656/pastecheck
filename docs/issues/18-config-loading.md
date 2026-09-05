# Title

TOML config loading, precedence, and validation

## Summary

Implement `src/config.rs` per DESIGN §11: path resolution, the full config
schema, precedence (CLI > file > defaults), validation with the safety
invariants, and the gate-mode fallback behavior (SEC-10).

## Context

Config is user-trusted but parsed defensively (boundary B3). Two behaviors
are security-relevant: a broken config must not brick paste (gate falls back
to defaults + warning) and must not silently disable core rules (PC220/PC302
floor at warning; PC220 survives `control=false`).

## Scope

- `src/config.rs` producing `EffectiveConfig` (extends the minimal struct
  from issue 05); wiring into all subcommands; unit + integration tests.

## Detailed Requirements

1. Path resolution order: `--config` > `PASTECHECK_CONFIG` >
   `$XDG_CONFIG_HOME/pastecheck/config.toml` >
   `~/.config/pastecheck/config.toml`. Missing file at the resolved default
   paths ⇒ defaults silently; missing file at an **explicitly given** path
   (flag/env) ⇒ `ConfigInvalid` (exit 3).
2. Schema exactly per §11 (keys, enums, defaults). Parse into a
   `RawConfig` with `serde(deny_unknown_fields)` **disabled** — unknown keys
   instead collected and warned to stderr (`pastecheck: warning: unknown
   config key "x"`), then ignored (SEC-10).
3. Validation errors (bad enum, ratio outside 0.0–1.0, unknown rule code in
   `[rules]`, malformed codepoint in `[allowlist]` — format `U+XXXX`):
   message names key + value + expected form.
4. Safety invariants: `[rules]` values below `warning` for PC220/PC302 ⇒
   validation error; `detectors.control = false` still leaves PC220 active
   (input-emitted).
5. Behavior on invalid config: `check`/`sanitize`/`rules` ⇒ exit 3;
   `gate` ⇒ stderr warning + built-in defaults (SEC-10) — implemented here
   as a `Config::load_for_gate()` variant returning
   `(EffectiveConfig, Vec<Warning>)`.
6. Precedence: every CLI flag with a config counterpart overrides it
   (`--fail-on`, `--max-bytes`, `--color`, `--enable/--disable`, sanitize
   strategy flags). Table-driven test proving each pair.

## Acceptance Criteria

- [ ] Default load with no file yields documented defaults (assert full
      struct).
- [ ] `[rules] "PC212" = "off"` suppresses TAB findings end-to-end
      (integration via `check`).
- [ ] `[allowlist] codepoints = ["U+00AD"]` suppresses a SHY-only finding.
- [ ] `"PC220" = "info"` ⇒ exit 3 with the invariant message.
- [ ] Unknown key warns but proceeds; `gate` with syntactically broken TOML
      proceeds on defaults with warning (unit test on `load_for_gate`).
- [ ] `PASTECHECK_CONFIG=/nonexistent pastecheck check` ⇒ exit 3;
      default-path absence ⇒ exit per findings.
- [ ] Precedence table test passes for all flag/key pairs.

## Validation

```sh
cargo test --locked config::
cargo test --locked --test cli -- config_
```

## Dependencies

Blocked by: 04, 13.

## Non-goals

Hook env vars (`PASTECHECK_GATE_FAIL` etc. are snippet-side — 21/22),
config hot-reload, system-wide config paths.

## Design References

- `docs/DESIGN.md` §11 (normative schema), §3.3 SEC-10
