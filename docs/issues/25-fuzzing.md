# Title

Fuzz targets and scheduled fuzz CI

## Summary

Add cargo-fuzz with three targets per DESIGN §16.6 — `engine` (no-panic),
`ansi` (parser robustness), `sanitize` (property-asserting) — seeded from
the corpus, plus a weekly scheduled CI job.

## Context

The ANSI parser and the sanitizer rebuild logic are the two components most
likely to harbor edge-case bugs against adversarial input (SEC-4, SEC-5).
Fuzzing complements the corpus with what nobody thought to write down.

## Scope

- `fuzz/` directory (`cargo fuzz init` layout), 3 targets, seed corpus
  generation script, `.github/workflows/fuzz.yml`.

## Detailed Requirements

1. Targets (each takes `&[u8]`, converts via `from_utf8_lossy` where the API
   needs `&str` — the raw-bytes entry also exercises PC220 paths through
   `input::` helpers):
   - `engine`: run full analysis with default config; assert no panic and
     that `Report` invariants hold (spans ascending & within text bounds;
     counts ≥ span lens; max_severity consistent).
   - `ansi`: run the layer-1 parser; assert no panic, consumed ranges
     sorted/non-overlapping/in-bounds, parser terminates (implicit).
   - `sanitize`: run sanitize with each strategy combination from a byte of
     the input (deterministic mapping); assert §10.3 properties: idempotent,
     effective for strip/map categories, no new non-target codepoints.
2. Seeds: a build script/`xtask`-style test that decodes every corpus case
   input into `fuzz/corpus/<target>/` (escaped TOML → raw bytes only inside
   gitignored fuzz corpus dirs; seeds are generated, not committed).
3. `fuzz.yml`: `schedule: weekly` + `workflow_dispatch`; ubuntu-latest;
   nightly toolchain (cargo-fuzz requirement) installed pinned; run each
   target `-max_total_time=600`; upload crash artifacts; job is
   non-blocking for PRs (schedule-only) but opens visibility via job
   summary. Repository crash triage rule documented in `fuzz/README.md`:
   a reproducing crash becomes a corpus case in the same PR as its fix.
4. `cargo fuzz build` must compile in default CI? No — nightly-only; keep
   fuzz out of `ci.yml` (documented in fuzz/README.md).

## Acceptance Criteria

- [ ] `cargo +nightly fuzz run engine -- -max_total_time=60` runs clean
      locally (and similarly for `ansi`, `sanitize`).
- [ ] Seed generation produces ≥ 1 seed per corpus case.
- [ ] Deliberately injected panic (local experiment, not committed) is found
      by the `engine` target within 60 s — proves the harness bites.
- [ ] `fuzz.yml` actions SHA-pinned, `permissions: contents: read`.
- [ ] `fuzz/README.md` documents run/triage/regression workflow.

## Validation

```sh
cargo +nightly fuzz build
cargo +nightly fuzz run ansi -- -max_total_time=60
ls fuzz/corpus/engine | head
```

## Dependencies

Blocked by: 05, 06, 07, 08, 09, 10, 11, 12, 17.

## Non-goals

Coverage-guided differential fuzzing against other tools; OSS-Fuzz
onboarding (worth revisiting post-v1).

## Design References

- `docs/DESIGN.md` §16.6, §10.3, §3.3 SEC-4, SEC-5
