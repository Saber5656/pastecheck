# Title

Release workflow (4 targets) and RELEASING.md

## Summary

Add `.github/workflows/release.yml` per DESIGN §17 and ADR-006: tag-driven
native builds for the 4-target matrix, tar.gz artifacts + SHA256SUMS, draft
GitHub Release — plus `RELEASING.md` documenting the manual crates.io
publish and the tag procedure.

## Context

The release pipeline is part of the security posture (SEC-13): explicit,
SHA-pinned, least-privilege, no generator tooling, no registry tokens in CI.

## Scope

- `release.yml`, `RELEASING.md`, `CHANGELOG.md` bootstrap (Keep a Changelog
  skeleton with `Unreleased` section).

## Detailed Requirements

1. Trigger: `push: tags: ["v*"]`. Top-level `permissions: {}`.
2. Job `build` (matrix, `permissions: contents: read`):

   | target | runner | notes |
   |---|---|---|
   | x86_64-apple-darwin | macos-13 | |
   | aarch64-apple-darwin | macos-14 | |
   | x86_64-unknown-linux-musl | ubuntu-24.04 | apt `musl-tools` |
   | aarch64-unknown-linux-musl | ubuntu-24.04-arm | apt `musl-tools` |

   Steps: checkout (SHA-pinned), install pinned toolchain + target,
   `cargo build --release --locked --target <t>`, verify version match
   (`[ "v$(cargo pkgid | sed 's/.*#//')" = "$GITHUB_REF_NAME" ]` — fail on
   mismatch), `tar czf pastecheck-<ver>-<target>.tar.gz` containing the
   binary + `LICENSE-MIT` + `LICENSE-APACHE` + `README.md`, upload artifact.
3. Job `publish` (needs build; the **only** job with `permissions:
   contents: write`): download artifacts, generate single `SHA256SUMS`
   (`sha256sum *.tar.gz`), create **draft** GitHub Release for the tag with
   artifacts + SHA256SUMS attached and CHANGELOG section as body. Human
   publishes the draft (merge ≠ release policy).
4. Smoke check in `build`: run the built binary (`--version`, and
   `printf 'ls\n' | pastecheck; test $? -eq 1`) on each runner — native
   targets make this possible everywhere.
5. `RELEASING.md`: exact steps — update CHANGELOG, bump `Cargo.toml`
   version, PR, merge, `git tag vX.Y.Z && git push origin vX.Y.Z`, publish
   draft Release after checking artifacts, then manual
   `cargo publish --locked` (with a note that the crates.io token is
   entered by the maintainer locally and never stored in the repo/CI —
   owner's credential policy).

## Acceptance Criteria

- [ ] Tagging `v0.1.0-rc.1` on a test branch produces a draft release with
      4 tarballs + SHA256SUMS; checksums verify (`sha256sum -c`).
- [ ] Version-mismatch guard fails the workflow when tag ≠ Cargo.toml
      version (test with a deliberate mismatch tag, then delete it).
- [ ] Extracted binaries run on a clean macOS (arm64) and Linux
      (x86_64 container without glibc — musl static verified via
      `ldd` reporting "not a dynamic executable").
- [ ] No `uses:` tag references; only SHAs. Only `publish` has write perms.
- [ ] `RELEASING.md` executable by someone who has never released this repo.

## Validation

```sh
grep -nE 'uses:.*@[a-f0-9]{40}' .github/workflows/release.yml
! grep -nE 'uses:.*@v[0-9]' .github/workflows/release.yml
grep -n 'contents: write' .github/workflows/release.yml | wc -l   # exactly 1
# then: push an rc tag and verify the draft release end-to-end
```

## Dependencies

Blocked by: 02.

## Non-goals

crates.io automation (manual by policy), Homebrew tap (v2, separate repo),
provenance attestation (v2), installers.

## Design References

- `docs/DESIGN.md` §17, §3.3 SEC-13
- `docs/decisions/ADR-006-release-engineering.md`
- `docs/research/03-rust-crates-and-release.md` §2–3
