# Title

pty-driven shell E2E test harness (zsh + bash)

## Summary

Build the `expect`-based E2E harness of DESIGN §16.5: drive real interactive
`zsh -f` and `bash --norc` sessions on a pty, send literal bracketed-paste
frames, and assert the full hook behavior matrix; wire it as a dedicated CI
job.

## Context

Unit tests cannot prove the only thing that matters for the hooks: that a
real shell on a real pty, receiving real `ESC[200~ … ESC[201~` frames, ends
up with the right buffer contents. Flakiness is a known unknown (U4) — the
harness design must be deterministic-first.

## Scope

- `tests/e2e/run-zsh.exp`, `tests/e2e/run-bash.exp`, a `tests/e2e/run.sh`
  orchestrator, and a `shell-e2e` CI job appended to `ci.yml`.

## Detailed Requirements

1. Harness rules (determinism): every `expect` waits on explicit sentinel
   strings (unique `PS1` set by the harness, e.g. `PC_E2E>`), never on
   timing; global timeout 20 s per scenario; each scenario runs in a fresh
   shell process; the built binary path is injected via `PASTECHECK_BIN`.
2. Scenarios, each for zsh and bash (intercept mode), asserting the final
   line buffer (echoed via a harness keystroke, e.g. `C-a "OK:" Enter`
   after decision) and shell survival:
   - S1 clean paste `echo hi` ⇒ inserted unchanged, no prompt.
   - S2 `ls␊` (trailing LF) ⇒ prompt shown; key `a` ⇒ buffer empty.
   - S3 same paste; key `p` ⇒ buffer contains `ls` + newline in buffer
     without execution (zsh) / spliced line (bash).
   - S4 `cu<U+200B>rl x` (ZWSP sent as raw bytes by the harness) ⇒ key `s`
      ⇒ buffer `curl x`.
   - S5 `PASTECHECK_DISABLE=1` ⇒ hostile paste inserts with no prompt.
   - S6 `PASTECHECK_BIN=/nonexistent` ⇒ paste inserts, warning shown
      (fail-open); with `PASTECHECK_GATE_FAIL=closed` ⇒ discarded.
   - S7 6 MiB paste ⇒ rc=3 limited prompt path (p/a only) — zsh only
      (bash cap behavior asserted at unit level; pty transfer of 6 MiB is
      slow/flaky).
   - S8 bash keybind mode: clipboard faked via PATH shim; `C-x C-v` with
      dirty clipboard prompts; `s` inserts sanitized.
3. CI job `shell-e2e`: ubuntu-latest installs `zsh expect` (apt) and runs
   both suites against the release-profile binary; macos-latest runs the zsh
   suite (`expect` preinstalled). Job is **required** (not
   continue-on-error); flake mitigation = deterministic waits + 2 automatic
   retries per scenario inside `run.sh` with retry count reported.
4. Failure output: on assertion failure, dump the full pty transcript with
   non-printables escaped (reuse `pastecheck` itself? no — `cat -v` is
   sufficient) so CI logs are actionable.

## Acceptance Criteria

- [ ] All scenarios pass locally on macOS (zsh) and in a Linux container
      (zsh + bash) 10/10 consecutive runs.
- [ ] CI job green on both OSes; retry counter 0 in the normal case.
- [ ] Removing the hook install line makes S2 fail (harness actually tests
      the hook — negative control committed as a harness self-test).
- [ ] Runtime of the full E2E job ≤ 5 minutes.

## Validation

```sh
cargo build --release
tests/e2e/run.sh --shell zsh --bin target/release/pastecheck
tests/e2e/run.sh --shell bash --bin target/release/pastecheck
for i in $(seq 10); do tests/e2e/run.sh --all || exit 1; done
```

## Dependencies

Blocked by: 21, 22.

## Non-goals

tmux/ssh matrix testing (documented manually, SEC-15), fish, performance
measurement (24).

## Design References

- `docs/DESIGN.md` §16.5, §13 (behavior under test)
- ISSUE_PLAN §8 U4 (flakiness contingency)
- `docs/research/02-shell-paste-interception.md` §1 (frame format)
