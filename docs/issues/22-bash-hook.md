# Title

bash hook snippet (intercept + keybind modes) and `shell-init bash`

## Summary

Implement `src/shell/snippets/pastecheck.bash` with the two modes of DESIGN
§13.2 — default `intercept` (rebind `\e[200~`, manual read to `\e[201~`) and
fallback `keybind` (`C-x C-v` guarded clipboard insert) — plus version
gating and the shared gate flow.

## Context

readline has no paste hook (research 02 §3); interception re-implements
paste collection in shell code and is the least-stable component in the
product (known unknown U1). The dual-mode design contains that risk.

## Scope

- The `.bash` snippet; `shell-init bash` wiring; scripted non-pty unit tests
  (pty E2E in issue 23).

## Detailed Requirements

1. Install block: require bash ≥ 4.4 (`BASH_VERSINFO`); on older, print
   notice and install nothing. Locate binary as in issue 21. Mode from
   `PASTECHECK_BASH_MODE` ∈ {intercept (default), keybind}; auto-fallback to
   keybind with stderr notice if `enable-bracketed-paste` is unsupported.
2. Intercept mode:
   - `bind 'set enable-bracketed-paste on'`;
     `bind -x '"\e[200~": __pastecheck_paste'`.
   - Capture loop: `LC_ALL=C IFS= read -r -s -N 1 -t 0.5 ch </dev/tty`
     accumulating until literal `ESC[201~`; strip the terminator; hard cap
     5 MiB (on cap: stop reading, treat captured as the paste, warn);
     timeout without terminator ⇒ treat captured bytes as the paste
     (research 02 §3; SEC-15-adjacent, documented).
   - NUL caveat: bash drops NUL in variables — accepted risk, noted in
     snippet comment and README (research 02 §3).
3. Gate flow (both modes): pipe content via `printf '%s' "$content" |
   "$bin" gate --shell bash`; rc drives the same decision table as zsh
   (§13.1 rows 3–10) with prompts written to `/dev/tty` and
   `read -r -s -n 1 </dev/tty`; sanitize path uses the `printf x` sentinel
   capture; insert = splice into `READLINE_LINE` at `READLINE_POINT`,
   advancing the point by `${#content}`.
4. Keybind mode: `bind -x '"\C-x\C-v": __pastecheck_clipboard_paste'`,
   normative flow per DESIGN §13.2: fetch the clipboard bytes **once** into
   a variable using the same tool order as DESIGN §12
   (`pbpaste` / `wl-paste --no-newline` /
   `xclip -selection clipboard -o` / `xsel --clipboard --output`; capture
   with the `printf x` sentinel for trailing-newline fidelity; no tool
   available ⇒ stderr notice, do nothing); then run the standard gate flow
   on that variable via stdin and apply the same p/a/s prompt and READLINE
   splice as intercept mode. Single fetch = no TOCTOU between analyzed and
   inserted content. The small duplicated tool-selection block is accepted
   by design (DESIGN §13.2).
5. SEC-8 rules identical to zsh: stdin pipes only, no eval, no temp files,
   `_pastecheck_`/`__pastecheck_` namespacing, no content in argv/env.
6. `shell-init bash` prints the snippet byte-exact; golden test.

## Acceptance Criteria

- [ ] `bash --norc -c 'eval "$(pastecheck shell-init bash)"; declare -F'`
      lists the hook functions; bash 4.3 simulation (env var override in
      test harness) installs nothing and prints the notice.
- [ ] `bash -n` passes on the snippet.
- [ ] Terminator scan unit test (scripted, no pty): function fed
      `abc$'\e[201~'` byte stream extracts `abc`; embedded partial marker
      `a$'\e[201'b$'\e[201~'` extracts `a\e[201b`.
- [ ] Splice test: given `READLINE_LINE=xy`, `READLINE_POINT=1`, inserting
      `Z` yields `xZy`, point 2 (function-level test via `bind -x` shim).
- [ ] Keybind fetch test (scripted, fake `pbpaste`/`wl-paste` shims on
      PATH): fetch preserves trailing newline (sentinel pattern) and selects
      tools in the §12 order; missing tools produce the notice and no
      insertion.
- [ ] Grep-checks as in issue 21 (no eval, namespacing, no content-in-argv).
- [ ] Full pty scenarios in issue 23.

## Validation

```sh
cargo test --locked shell::bash
bash -n src/shell/snippets/pastecheck.bash
! grep -n 'eval' src/shell/snippets/pastecheck.bash
```

## Dependencies

Blocked by: 17, 20.

## Non-goals

zsh (21), pty E2E (23), readline < 7.0 support, NUL fidelity (accepted
risk).

## Design References

- `docs/DESIGN.md` §13.2, §13.3, §3.3 SEC-8, SEC-9, SEC-15
- `docs/research/02-shell-paste-interception.md` §3
- ISSUE_PLAN §8 U1 (contingency)
