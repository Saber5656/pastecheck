# Title

CI workflow: fmt, clippy, test matrix, and MSRV check

## Summary

Add `.github/workflows/ci.yml` running formatting, lints, tests on
ubuntu-latest + macos-latest, and an MSRV check — with SHA-pinned actions and
least-privilege permissions per SEC-12.

## Context

CI must be green from the first functional PR so every later issue inherits
the gates. The shell-E2E job is added later by issue 23; this issue creates
the workflow it will extend.

## Scope

- `ci.yml` only. No release workflow (26), no cargo-deny job (03).

## Detailed Requirements

1. Triggers: `pull_request` and `push` to `main`.
2. Top level: `permissions: {}`; each job sets `permissions: contents: read`.
   `concurrency` group per ref with `cancel-in-progress: true`.
3. Jobs:
   - `fmt`: `cargo fmt --all --check` (ubuntu-latest).
   - `clippy`: `cargo clippy --all-targets --locked -- -D warnings`
     (ubuntu-latest).
   - `test`: matrix `os: [ubuntu-latest, macos-latest]`;
     `cargo test --locked --all-targets`.
   - `msrv`: read `rust-version` from `Cargo.toml`, install exactly that
     toolchain, `cargo check --locked`.
4. Toolchain install via `actions/checkout` + `rustup` shell commands or a
   well-known action — **every** third-party action referenced by full commit
   SHA with a trailing `# vX.Y.Z` comment.
5. Cache: `~/.cargo/registry`, `~/.cargo/git`, `target/` keyed on
   `Cargo.lock` hash + toolchain + os (use `actions/cache`, SHA-pinned).
6. Workflow must fail if any job fails (no `continue-on-error`).

## Acceptance Criteria

- [ ] CI runs on a PR touching any file and all four jobs pass on the
      scaffolding from issue 01.
- [ ] Grep check: no `uses:` line references a mutable tag
      (`@v4`-style) — SHAs only.
- [ ] No job has write permissions.
- [ ] MSRV job installs the `rust-version` toolchain, not stable.

## Validation

```sh
# static checks locally
grep -nE 'uses:.*@[a-f0-9]{40}' .github/workflows/ci.yml   # every uses matches
! grep -nE 'uses:.*@v[0-9]' .github/workflows/ci.yml
# then: open a draft PR and observe all jobs green
```

## Dependencies

Blocked by: 01.

## Non-goals

cargo-deny / Dependabot (03), shell E2E job (23), release workflow (26).

## Design References

- `docs/DESIGN.md` §17 (CI), §3.3 SEC-11, SEC-12
- `docs/research/03-rust-crates-and-release.md` §2–3
