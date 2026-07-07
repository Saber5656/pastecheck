# ADR-005: Clipboard access via system utilities, read-only

Status: accepted (2026-07-08)

## Context

`--clipboard` needs clipboard *read* on macOS and Linux (X11 + Wayland).
Options: a native clipboard crate (e.g. `arboard`) or invoking the platform's
standard utilities (`pbpaste`, `wl-paste`, `xclip`/`xsel`).

## Decision

Invoke system utilities with fixed argument vectors, no shell, output cap,
and a 5 s timeout (DESIGN §12, SEC-7): `pbpaste` (macOS),
`wl-paste --no-newline` (Wayland), `xclip -selection clipboard -o` with
`xsel --clipboard --output` fallback (X11). Absent tools produce exit 3 with
an install hint. pastecheck never **writes** the clipboard in v1 by any
mechanism (SEC-6).

## Alternatives considered

- **`arboard`**: portable API, but pulls native X11/Wayland/AppKit binding
  stacks into a security tool that only needs read; larger audit surface and
  binary. Rejected for v1; reconsider if utility fragmentation on Wayland
  becomes a real support burden (known unknown).
- **Direct protocol implementations**: far too much surface for v1.

## Consequences

- Zero clipboard-related compile-time dependencies; the trust moves to
  user-installed binaries resolved via PATH — same trust level as the shell
  session itself (threat model §3.4).
- Linux users without `wl-paste`/`xclip`/`xsel` must install one (clear
  error message; documented in README).
- The clipboard adapter is trivially fakeable in tests (fake executables on
  PATH), which issue 18 exploits.
