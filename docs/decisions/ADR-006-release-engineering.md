# ADR-006: Hand-rolled, SHA-pinned release workflow; manual crates.io publish

Status: accepted (2026-07-08, distribution scope confirmed with product
owner: GitHub Releases + crates.io; Homebrew tap deferred to v2)

## Context

v1 ships prebuilt binaries for 4 targets plus `cargo install`. Release
tooling options: `cargo-dist`/`dist` (workflow generator) vs an explicit
hand-written GitHub Actions workflow. A security tool's release pipeline is
itself an attack surface (SEC-12, SEC-13).

## Decision

1. **Hand-rolled `release.yml`** (~100 lines, DESIGN §17): tag `v*` trigger;
   native builds on macos-13 (x86_64-darwin), macos-14 (aarch64-darwin),
   ubuntu-24.04 (x86_64-musl), ubuntu-24.04-arm (aarch64-musl — GA arm64
   runners, free for public repos, verified in research 03); `--locked`
   builds; tar.gz + single `SHA256SUMS`; draft GitHub Release.
2. All actions pinned to full commit SHAs; workflow-level `permissions: {}`;
   only the release job gets `contents: write`.
3. **crates.io publish stays manual** (`cargo publish --locked` by the
   maintainer per `RELEASING.md`): no registry token stored in CI, matching
   the repository owner's policy that credentials are handled by the human.
4. Merge ≠ release: main can advance freely; releases happen only via tags.

## Alternatives considered

- **cargo-dist / dist**: generates all of the above plus installers, but
  inserts a generator between the repo and its release process and has
  changed stewardship; for 4 targets the generated complexity exceeds the
  hand-written equivalent. Deferred — revisit at v2 alongside the Homebrew
  tap (which is its main value-add).
- **Automated crates.io publish on tag**: convenient, but requires a
  long-lived secret in CI. Rejected for v1.
- **cross/QEMU for aarch64-linux**: unnecessary since arm64 hosted runners
  are GA. Rejected.

## Consequences

- Every release step is readable in one YAML file; auditors need no tool
  knowledge.
- Release provenance/attestation is not in v1 (deferred; checksums only).
- Dependabot keeps pinned action SHAs current via PRs.
