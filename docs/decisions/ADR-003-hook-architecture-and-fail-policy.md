# ADR-003: Thin shell hooks over a `gate` subcommand; fail-open default

Status: accepted (2026-07-08)

## Context

The guard must run inside zsh/bash on every paste. Analysis logic in shell
script is untestable and unsafe (arbitrary-byte handling in shell is
minefield territory); but a hook that can block all pasting when broken will
be ripped out by users, destroying the product's protective value.

## Decision

1. **All analysis lives in the Rust binary.** Snippets only: capture the
   bracketed paste, pipe it to `pastecheck gate` via stdin, display the
   pre-escaped plain-ASCII report, read one decision key, insert/discard/
   re-pipe-to-`sanitize`. Stable `gate` contract: exit 0 = insert, 1 =
   prompt, 3 = could-not-scan → limited prompt (DESIGN §8.5, §13).
2. **Failure policy (SEC-9)**: scan-impossible (exit 3) is never a silent
   pass — the user is prompted with the reason. Binary missing/crashed
   defaults to **fail-open with a visible warning**; `PASTECHECK_GATE_FAIL=closed`
   opts into strict blocking.
3. Content transfer is stdin-only (never argv/env/tmpfiles — SEC-8), and
   snippets are embedded in the binary (`shell-init`) so snippet and binary
   version can never skew.

## Alternatives considered

- **Analysis in shell (pure-snippet product)**: unauditable, byte-mangling,
  duplicated per shell; rejected.
- **Fail-closed default**: a deleted/broken binary would make paste unusable;
  field experience says users then remove the hook permanently. Fail-open
  with a loud warning keeps them protected long-term. Rejected as default,
  kept as opt-in.
- **Long-lived daemon queried by hooks**: lower per-paste spawn cost but adds
  a persistent process, IPC boundary, and lifecycle management; unnecessary
  at measured spawn+scan latency (DESIGN §15). Rejected for v1.

## Consequences

- One protocol to test (gate), three consumers (zsh, bash, E2E harness).
- The prompt UX is limited to what `zle -M`/`/dev/tty` can do — accepted.
- Exit-code contract becomes public API for anyone writing hooks for other
  shells (fish in v2 reuses it unchanged).
