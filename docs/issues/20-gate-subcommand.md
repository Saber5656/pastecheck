# Title

`gate` subcommand: the hook-facing protocol

## Summary

Implement `pastecheck gate` per DESIGN §8.5: stdin in, compact plain-ASCII
pre-escaped report out, exit codes 0/1/3, `hook.show_max_findings` cap, and
the latency budget — the stable contract both shell snippets consume.

## Context

The gate is boundary B6's binary side. Its output renders inside `zle -M`
and raw `/dev/tty` writes, so it must be pure ASCII (strict escaping
variant), uncolored, and line-capped. Its exit codes drive the hook state
machines (ADR-003).

## Scope

- Gate implementation in `cli.rs`/`render` (compact formatter);
  `--shell zsh|bash` flag (reserved for shell-specific tailoring; v1 output
  identical — flag validated and recorded); integration tests.

## Detailed Requirements

1. Input: stdin bytes through the standard §6 pipeline with gate's config
   (`load_for_gate`, SEC-10 fallback).
2. Output (stdout): nothing when clean. With findings ≥ `fail_on`: header
   `pastecheck: N findings (X critical, Y warning) in L lines`, then up to
   `hook.show_max_findings` (default 10) lines, each
   `  <severity> <code> <name> <first-offender> ×<count> @ line <L>`
   using `escape_str_ascii` for anything input-derived, then
   `  ... and M more` when capped. No SGR, no non-ASCII, ≤ 79 cols per line
   (truncate with `...`).
3. Exit codes: 0 clean; 1 findings ≥ `fail_on`; 3 could-not-scan
   (size cap / IO / clipboard-N/A never applies — stdin only). Usage errors
   (2) only for invalid flags.
4. Info findings never trigger exit 1 and never render in gate output.
5. Latency: `#[ignore]`d test asserting 64 KiB hostile input completes the
   full process (spawn via assert_cmd) within a generous CI-safe bound
   (500 ms) and records actual time to stdout for the nightly perf job;
   the §15 25 ms scan budget is checked as a library-level timing test.
6. Stderr: reserved for warnings (config fallback) and error messages —
   single line each, no input content (SEC-2).

## Acceptance Criteria

- [ ] `printf 'ls' | pastecheck gate` ⇒ exit 0, empty stdout.
- [ ] `printf 'ls\n' | pastecheck gate` ⇒ exit 1; output contains
      `PC105` and `@ line 1`; pure ASCII (test asserts every output byte
      < 0x80 and not a control except `\n`).
- [ ] 15-finding input with default config ⇒ 10 lines + `... and 5 more`.
- [ ] 6 MiB input ⇒ exit 3, stderr has the §6.2 message, stdout empty.
- [ ] Broken config + `gate` ⇒ exit per findings with stderr warning
      (SEC-10), not exit 3.
- [ ] `--shell fish` ⇒ exit 2 (only zsh|bash accepted in v1).
- [ ] Output identical for `--shell zsh` and `--shell bash` (golden).

## Validation

```sh
cargo test --locked --test cli -- gate_
printf 'a\xe2\x80\xaeb\n' | cargo run -q -- gate; echo "exit=$?"   # a + U+202E + b + LF
```

## Dependencies

Blocked by: 13, 14.

## Non-goals

Snippets themselves (21/22), prompting/keys (hook-side), sanitize (17).

## Design References

- `docs/DESIGN.md` §8.5 (normative), §9.1 strict variant, §13.3, §15
- `docs/decisions/ADR-003-hook-architecture-and-fail-policy.md`
