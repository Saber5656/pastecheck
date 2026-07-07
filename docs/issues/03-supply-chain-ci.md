# Title

Supply-chain gates: cargo-deny and Dependabot

## Summary

Add `deny.toml` with a `cargo-deny` CI job (advisories, license allowlist,
bans, sources) and `.github/dependabot.yml` for cargo + github-actions
ecosystems, per SEC-12.

## Context

pastecheck is a security tool; dependency risk is part of the product threat
model (DESIGN §3.2 B7). Gates must exist before the dependency tree grows in
Wave 1+.

## Scope

- `deny.toml`, a `deny` job appended to `ci.yml`, `dependabot.yml`.

## Detailed Requirements

1. `deny.toml`:
   - `[advisories]`: deny on all advisories (`vulnerability`, `unmaintained`,
     `unsound`, `notice` at minimum warn; vulnerabilities = error).
   - `[licenses]`: allowlist exactly `MIT`, `Apache-2.0`,
     `Apache-2.0 WITH LLVM-exception`, `Unicode-3.0`, `BSD-3-Clause`,
     `Zlib`. Anything else fails (extend only via PR review).
   - `[bans]`: `multiple-versions = "warn"`; explicitly deny crates that
     imply network or unsafe surface creep: `openssl-sys`, `native-tls`,
     `reqwest`, `hyper`, `tokio` (SEC-1 tripwire).
   - `[sources]`: allow only crates.io (`unknown-registry = "deny"`,
     `unknown-git = "deny"`).
2. CI job `deny` (ubuntu-latest, `permissions: contents: read`): install
   pinned cargo-deny version (record version in workflow comment), run
   `cargo deny check`.
3. `dependabot.yml`: `package-ecosystem: cargo` (weekly) and
   `github-actions` (weekly), each with a `groups:` entry batching minor/patch
   updates.

## Acceptance Criteria

- [ ] `cargo deny check` passes locally and in CI on the current tree.
- [ ] Adding a GPL crate to `Cargo.toml` makes `cargo deny check licenses`
      fail (verify locally, do not commit).
- [ ] Dependabot config validates (GitHub UI shows both ecosystems active).

## Validation

```sh
cargo deny check
cargo deny check licenses -- 2>&1 | grep -i allow   # allowlist active
```

## Dependencies

Blocked by: 01.

## Non-goals

Release workflow permissions (26); vendoring Unicode data (v2 / U3).

## Design References

- `docs/DESIGN.md` §3.3 SEC-1, SEC-12; §4.3
- `docs/research/03-rust-crates-and-release.md` §1, §3
