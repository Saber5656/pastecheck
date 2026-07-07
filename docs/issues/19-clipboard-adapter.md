# Title

Clipboard adapters (`--clipboard`) via system utilities

## Summary

Implement `src/clipboard.rs` per DESIGN §12 and ADR-005: platform detection,
fixed-argv invocation of `pbpaste` / `wl-paste` / `xclip` / `xsel` with
timeout and size cap, and precise exit-3 errors with install hints.

## Context

Boundary B2: hostile content through semi-trusted local tools. SEC-7 pins
the allowlist, argv, timeout, and cap; SEC-6 forbids any clipboard write.

## Scope

- `src/clipboard.rs`; `--clipboard` wiring for `check` and `sanitize`;
  integration tests using fake executables on a controlled PATH.

## Detailed Requirements

1. Selection logic exactly per §12: macOS ⇒ `pbpaste`; Linux with
   `WAYLAND_DISPLAY` ⇒ `wl-paste --no-newline`; else with `DISPLAY` ⇒
   `xclip -selection clipboard -o`, fallback `xsel --clipboard --output`
   when xclip is absent; neither var ⇒ `ClipboardUnavailable` with the §12
   message.
2. Invocation (SEC-7): `std::process::Command`, fixed argv, `stdin(null)`,
   capture stdout up to `max_bytes + 1` (beyond ⇒ `SizeExceeded` semantics
   identical to §6.2), kill after 5 s wall timeout (thread + `try_wait`
   loop or `wait_timeout`-style hand-rolled loop — no new dependency),
   stderr captured and included (escaped, first line only) in error text.
3. Spawn failure (`ENOENT`) ⇒ `ClipboardUnavailable`, message includes tool
   name and hint: macOS `pbpaste should always exist`; Linux
   `install wl-clipboard (Wayland) or xclip/xsel (X11)`.
4. Non-zero exit from the tool ⇒ `ClipboardUnavailable` with tool name +
   exit code.
5. Empty clipboard is valid (0 bytes ⇒ clean).
6. Bytes feed the standard §6 pipeline (UTF-8 policy, PC220, etc.).

## Acceptance Criteria

- [ ] Fake-PATH integration tests (scripts named `pbpaste`/`wl-paste`/
      `xclip`/`xsel` in a temp dir, `PATH` overridden): success path; tool
      exits 1; tool sleeps 10 s (timeout kills within ~5–6 s); output over
      cap ⇒ size error; ENOENT fallback xclip→xsel; no-DISPLAY error.
- [ ] Env selection: `WAYLAND_DISPLAY` beats `DISPLAY` (test both set).
- [ ] `pastecheck --clipboard` and `pastecheck sanitize --clipboard` both
      wired; conflict with FILE positional is a usage error (exit 2).
- [ ] Grep-check: no code path writes to any clipboard tool (`pbcopy`,
      `wl-copy`, `xclip -i` absent from the codebase).
- [ ] macOS manual smoke: `printf 'x\xe2\x80\x8by' | pbcopy && pastecheck
      --clipboard` (x + U+200B + y) reports PC301 (documented in PR as
      tested).

## Validation

```sh
cargo test --locked clipboard::
cargo test --locked --test cli -- clipboard_
```

## Dependencies

Blocked by: 13.

## Non-goals

Clipboard write (SEC-6, never in v1), native clipboard crate (ADR-005),
Windows.

## Design References

- `docs/DESIGN.md` §12, §3.3 SEC-6, SEC-7, §6.2
- `docs/decisions/ADR-005-clipboard-via-system-utilities.md`
