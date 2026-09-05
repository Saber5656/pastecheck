# Research: Shell Paste Interception Mechanisms (zsh, bash)

Status: reviewed input for `docs/DESIGN.md` §13 (shell integration) and issues 19–22.
Date: 2026-07-08. Mechanisms verified against zsh-users/zsh sources and GNU
readline documentation (links below).

## 1. Bracketed paste primer

Modern terminal emulators support *bracketed paste mode*: when the application
enables it (`ESC [ ? 2004 h`), any paste arrives on stdin wrapped as:

```
ESC [ 2 0 0 ~   <pasted bytes>   ESC [ 2 0 1 ~
```

This is the only portable signal that distinguishes "typed" from "pasted"
input, and it is where a paste guard must sit. Supported by Terminal.app,
iTerm2, kitty, Alacritty, WezTerm, GNOME Terminal, xterm, tmux (pass-through),
and mosh.

Limitation inherited by every consumer: the paste payload cannot itself
contain `ESC [ 2 0 1 ~`; terminals do not escape it. A paste that *contains*
the end marker will terminate capture early — the remainder arrives as typed
input. This is a known residual risk for all bracketed-paste consumers
(readline, zle, vim) and must be documented (DESIGN §13.3, SEC-15): the
remainder is still covered because it then goes through the shell's normal
self-insert path, and the truncated captured part is still scanned.

## 2. zsh (ZLE)

### Mechanism (verified)

- Since zsh 5.1, ZLE enables bracketed paste itself and dispatches the whole
  paste to the `bracketed-paste` widget (default: the `.bracketed-paste`
  builtin, which inserts the text into the buffer).
- **Key capability**: when a user-defined widget calls the builtin with a
  parameter name argument — `zle .bracketed-paste PASTED` — the pasted text is
  **assigned to that parameter instead of being inserted**. This is exactly
  how the stock `bracketed-paste-magic` function works
  (https://github.com/zsh-users/zsh/blob/master/Functions/Zle/bracketed-paste-magic).
- Therefore the pastecheck hook is:
  1. `zle -N bracketed-paste __pastecheck_paste` (replace the widget).
  2. Inside the widget: `local PASTED; zle .bracketed-paste PASTED` to capture.
  3. Analyze via `print -rn -- "$PASTED" | pastecheck gate --shell zsh`.
  4. On findings: display with `zle -M`, read one key with `read -k 1`.
  5. Insert by `LBUFFER+="$PASTED"` (or the sanitized replacement). Inserting
     text programmatically does **not** trigger `accept-line`, even when the
     text ends with a newline — the newline becomes part of the editing
     buffer. This is what makes "paste anyway"/"sanitize" safe to offer.

### Notes and edge cases

- zsh parameters can hold arbitrary bytes including NUL (zsh metafies
  internally), and `print -rn --` writes them back out verbatim; content
  fidelity through the variable round-trip is preserved.
- `$(...)` command substitution strips trailing newlines. When capturing
  sanitized output whose trailing newlines are significant, use the sentinel
  pattern: `out=$(cmd; printf x); out=${out%x}`.
- `zle -M` renders a message below the prompt but does not interpret ANSI
  colors reliably across versions — the `gate` output must be plain text
  (no SGR), pre-escaped by pastecheck itself.
- Widget conflicts: oh-my-zsh's `safe-paste` plugin and `bracketed-paste-magic`
  also replace the `bracketed-paste` widget. Chaining by *delegating* is not
  possible (the paste bytes can only be consumed once), so the pastecheck
  snippet must (a) detect that the current widget is not the builtin and warn,
  and (b) be installed last, replacing them. Documented behavior, not silent.
- Kill switch: the snippet must check `PASTECHECK_DISABLE=1` and a missing
  binary, and fall back to plain insertion (fail-open) with a one-time
  warning; a paste guard that can brick paste gets uninstalled.
- Latency: one process spawn per paste. A static Rust binary spawn+scan for
  typical paste sizes is a few milliseconds; budgeted in DESIGN §15.

## 3. bash (GNU readline)

### Constraints (verified)

- readline ≥ 7.0 (bash 4.4) supports `enable-bracketed-paste`; since bash 5.1
  it is **on by default**. When on, readline binds `ESC [ 2 0 0 ~` to
  `bracketed-paste-begin`, which collects until the end marker and inserts the
  paste as one string.
- readline has **no user hook** over the collected paste: there is no
  equivalent of zsh's "assign to parameter" call. Reference: GNU Readline
  manual, Init File Syntax (https://www.gnu.org/software/bash/manual/html_node/Readline-Init-File-Syntax.html).

### Primary mechanism: rebind the start marker

`bind -x '"\e[200~": __pastecheck_paste'` replaces the default
`bracketed-paste-begin` dispatch with a shell function. The function must then
read the paste body from the tty itself:

- Loop `IFS= read -r -s -N 1 -t <timeout> ch </dev/tty` accumulating into a
  variable, scanning for the literal terminator `ESC [ 2 0 1 ~`.
- Use a per-byte timeout (paste bytes arrive back-to-back; a gap means the
  terminator was lost) and a hard size cap as safety valves.
- After the gate decision, splice into the readline buffer:
  `READLINE_LINE="${READLINE_LINE:0:READLINE_POINT}${content}${READLINE_LINE:READLINE_POINT}"`
  and advance `READLINE_POINT` — the documented `bind -x` editing interface.
- Prompt UX: `bind -x` handlers run with the line redrawn afterwards; the
  decision prompt is written to `/dev/tty` and a single key read with
  `read -r -s -n 1 </dev/tty`.

Known limitations to encode in the design (DESIGN §13.2):

- bash variables cannot contain NUL; NUL bytes in a paste are dropped by
  `read`. Mitigation: the gate reads the original bytes via the pipe (before
  variable storage loses anything? No — the hook necessarily stores into a
  variable first). Accept and document: the bash hook cannot preserve NUL;
  since NUL is itself a Critical `control` finding and sanitize strips it,
  the practical impact is nil, but byte-fidelity for pathological pastes is
  weaker than zsh. Record as accepted risk.
- Requires bracketed paste enabled; the snippet enforces
  `bind 'set enable-bracketed-paste on'` and refuses to install (with a clear
  message) on bash < 4.4 / readline < 7.0.
- Multibyte reads with `read -N 1` operate on characters with the active
  locale; invalid UTF-8 in a paste can degrade capture. The gate itself
  handles invalid UTF-8 (PC220), but the bash capture loop must read bytes
  under `LC_ALL=C` to avoid locale-dependent behavior, then hand raw bytes to
  pastecheck.

### Fallback mechanism: verified-clipboard-paste keybinding

Because the primary mechanism is the least-stable part of the whole product
(it re-implements paste collection in shell code), the snippet also provides a
fallback mode selected by `PASTECHECK_BASH_MODE=keybind`:

- `C-x C-v` runs a `bind -x` function that reads the *system clipboard* via
  `pastecheck check --clipboard` / gate flow and inserts approved content into
  `READLINE_LINE`.
- This does not intercept raw terminal pastes but is deterministic on every
  bash ≥ 4.4 and gives bash users a reliable guarded path.

Mode selection, self-checks, and messaging are specified in issue 21.

## 4. tmux / ssh / mosh interplay

- tmux ≥ 3.x passes bracketed paste through to the inner application when the
  inner application enabled it; the hook works unchanged inside tmux.
- ssh is byte-transparent; the guard runs on the remote side only if installed
  there. Local hook + remote shell = unguarded (the local shell never sees the
  paste). Documented in user docs (issue 26).
- mosh supports bracketed paste since 1.3.

## 5. Consequences for the design

| Consequence | Where encoded |
|---|---|
| Guard logic must live in the binary; snippets stay thin (capture, one gate call, key prompt, insert) | ADR-003, DESIGN §13 |
| `gate` output must be plain, pre-escaped, bounded | DESIGN §8.5, §9.1 |
| Trailing-newline-preserving capture patterns required in snippets | DESIGN §13.3 |
| zsh: single-consumer widget conflict detection required | Issue 20 |
| bash: primary + fallback dual mode; version gating; LC_ALL=C byte reads | Issue 21 |
| E2E tests drive a real pty with real `ESC[200~ … ESC[201~` frames | Issue 22 |

## References

- zsh `bracketed-paste-magic` source — https://github.com/zsh-users/zsh/blob/master/Functions/Zle/bracketed-paste-magic
- zsh 5.1 bracketed paste background — https://archive.zhimingwang.org/blog/2015-09-21-zsh-51-and-bracketed-paste.html
- GNU Readline init file syntax (`enable-bracketed-paste`) — https://www.gnu.org/software/bash/manual/html_node/Readline-Init-File-Syntax.html
- readline bracketed-paste regression context — https://bugzilla.redhat.com/show_bug.cgi?id=1954366
