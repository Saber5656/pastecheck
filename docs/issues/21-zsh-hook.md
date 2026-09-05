# Title

zsh hook snippet and `shell-init zsh`

## Summary

Implement `src/shell/snippets/pastecheck.zsh` (embedded via `include_str!`)
and wire `pastecheck shell-init zsh`, implementing the widget state machine
of DESIGN §13.1 on top of the gate protocol.

## Context

The zsh mechanism is verified (research 02 §2): replace the
`bracketed-paste` widget, capture via `zle .bracketed-paste PASTED`, gate via
stdin pipe, prompt via `zle -M` + `read -k 1`, insert via `LBUFFER+=`.
Snippet code handles hostile bytes inside the user's interactive shell —
SEC-8 rules are hard requirements.

## Scope

- The `.zsh` snippet; `shell-init zsh` output (13's dispatch + this file);
  snippet-only unit tests via `zsh -f` scripted invocations of the pure
  helper functions (full pty E2E is issue 23).

## Detailed Requirements

1. Install block: locate binary (`PASTECHECK_BIN` override, else
   `command -v pastecheck`); if absent ⇒ print one-time stderr notice and
   define a pass-through widget (fail-open, SEC-9). If the current
   `bracketed-paste` widget is user-defined (not `.bracketed-paste`), warn
   once naming it (conflict policy, research 02 §2) and replace.
2. Widget `__pastecheck_paste` implementing §13.1 rows 1–11 exactly:
   - `PASTECHECK_DISABLE=1` or empty paste ⇒ plain insert, zero spawns.
   - Gate call: `out=$(print -rn -- "$PASTED" | "$__pastecheck_bin" gate
     --shell zsh 2>/dev/null)`; rc captured immediately.
   - rc=0 ⇒ insert; rc=1 ⇒ `zle -M` shows gate output + key prompt
     `[p]aste  [a]bort  [s]anitize  [d]etails`; rc=3 ⇒ limited prompt
     (p/a) showing the stderr line; other rc ⇒ `PASTECHECK_GATE_FAIL`
     (unset/`open` ⇒ insert + warning; `closed` ⇒ discard + warning).
   - Keys: `p` insert original; `a`/Ctrl-C/ESC discard with
     `zle -M "pastecheck: paste aborted"`; `s` ⇒ sanitize via
     `san=$(print -rn -- "$PASTED" | "$__pastecheck_bin" sanitize
     2>/dev/null; printf x)` + `san=${san%x}` (trailing-newline fidelity),
     insert `san`; sanitize spawn failure ⇒ back to prompt; `d` ⇒ show up to
     20 lines of `check --color never` output via `zle -M`, re-prompt; any
     other key ⇒ re-prompt (max 3 unknown keys ⇒ abort).
   - Insert = `LBUFFER+="$content"` (never `zle accept-line`).
3. SEC-8: content only ever flows via `print -rn -- "$PASTED" |` pipes; no
   argv, no env export, no temp files, no `eval`.
4. Namespacing: all functions/params `__pastecheck_*`; no aliases; no
   `setopt` leaks (use `emulate -L zsh` in functions).
5. `shell-init zsh` prints the snippet byte-exact (golden test) and exits 0.

## Acceptance Criteria

- [ ] `eval "$(pastecheck shell-init zsh)"` in `zsh -f` defines the widget:
      `zle -l | grep __pastecheck` and `bindkey` unchanged otherwise.
- [ ] Helper-level tests (scripted `zsh -f`, no pty): sentinel capture
      preserves trailing newline; disable-env short-circuits (assert via
      xtrace that no spawn happens); conflict warning fires when a dummy
      widget pre-exists.
- [ ] Snippet passes `zsh -n` (syntax) for zsh 5.8 and current stable.
- [ ] Grep-checks on the snippet: no `eval`, no `>`/`>>` redirection of
      content, no `$PASTED` in any argv position, all symbols prefixed.
- [ ] Full interactive scenarios covered by issue 23 (this issue lands the
      snippet + static/unit checks only).

## Validation

```sh
cargo test --locked shell::zsh
zsh -n src/shell/snippets/pastecheck.zsh
grep -c '__pastecheck_' src/shell/snippets/pastecheck.zsh   # namespacing present
! grep -n 'eval' src/shell/snippets/pastecheck.zsh
```

## Dependencies

Blocked by: 17, 20.

## Non-goals

bash (22), pty E2E (23), bracketed-paste-magic co-existence beyond
warn-and-replace (U2).

## Design References

- `docs/DESIGN.md` §13.1 (state machine normative), §13.3, §3.3 SEC-8, SEC-9
- `docs/research/02-shell-paste-interception.md` §2
- `docs/decisions/ADR-003-hook-architecture-and-fail-policy.md`
