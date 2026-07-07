# Title

CLI command tree, exit codes, and `rules` subcommand

## Summary

Implement `src/cli.rs` + `src/main.rs` with clap: the full command tree of
DESIGN §8.1 (`check` as default, `sanitize`, `gate`, `shell-init`, `rules`),
the §8.2 exit-code map, and the human `rules` listing. Subcommands whose
bodies land later dispatch to stubs returning a clear "not implemented"
operational error.

## Context

This issue freezes the user-facing contract (flags, defaults, exit codes) so
renderer/sanitizer/gate/hook issues plug into stable dispatch. Content is
never accepted via argv (SEC-2/B4): sources are stdin/FILE/clipboard only.

## Scope

- clap 4 (derive) dependency; `cli.rs`, `main.rs`; wiring `check` end-to-end
  (input → engine → placeholder plain renderer listing findings one-per-line
  as `severity code count` until issue 15 replaces it); `rules` human table.

## Detailed Requirements

1. Command tree and flags exactly per DESIGN §8.1, including `check` as
   default subcommand (`args_conflicts_with_subcommands` pattern), FILE
   positional accepting `-` for stdin, `--clipboard` conflicting with FILE.
2. Exit codes per §8.2 via a single `fn exit_code(result) -> i32` in
   `main.rs`; clap parse errors mapped to 2 (clap's default 2 preserved,
   `--help`/`--version` exit 0).
3. stdin-is-TTY without FILE/`--clipboard` ⇒ usage error (exit 2) with hint
   `reading from a terminal; pipe input or pass a file / --clipboard`.
4. `--fail-on` default `warning`; `--quiet` suppresses stdout; diagnostics
   on stderr only. `--enable`/`--disable` accept comma-separated category
   names, validated against the five categories (unknown ⇒ exit 2 naming the
   bad token).
5. `rules` (human): print the §5.3 table (code, category, default severity,
   name) from the registry — no hardcoded copies; include compiled-in
   versions line (`pastecheck X.Y.Z; unicode crates: …`) per §8.7.
6. Env: `PASTECHECK_CONFIG` read here and passed down (config parsing itself
   is issue 18; until then only `--max-bytes` and defaults apply).
7. `gate`, `shell-init`, `sanitize` subcommands parse their flags but return
   the stub error (exit 3, message `not implemented yet: <cmd>`) until their
   issues land.

## Acceptance Criteria

- [ ] `printf 'ls\n' | pastecheck` exits 1 (PC105 ≥ warning) with placeholder
      output; `printf 'ls' | pastecheck` exits 0.
- [ ] `pastecheck --fail-on critical` with warning-only input exits 0.
- [ ] `pastecheck nonexistent.txt` exits 3; `pastecheck --bogus` exits 2;
      TTY-stdin rule exits 2 (assert_cmd with no piped stdin).
- [ ] `--disable newline,control` skips those categories (verify via
      placeholder output); `--disable bogus` exits 2.
- [ ] `pastecheck rules` lists exactly 26 rows; `pastecheck rules | wc -l`
      stable.
- [ ] `--quiet` produces no stdout but correct exit code.
- [ ] Help texts exist for every flag (clap `--help` snapshot committed as a
      golden file).

## Validation

```sh
cargo test --locked --test cli
cargo run -q -- rules | head -5
```

## Dependencies

Blocked by: 05, 06.

## Non-goals

Human/JSON rendering (15/16), config file (18), gate behavior (20),
snippets (21/22), sanitize behavior (17).

## Design References

- `docs/DESIGN.md` §8.1–§8.3, §8.7, §14, §3.3 SEC-14
