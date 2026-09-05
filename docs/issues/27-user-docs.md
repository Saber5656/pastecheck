# Title

User documentation: README, SECURITY.md, CONTRIBUTING.md

## Summary

Rewrite `README.md` for the finished v1 (install, quickstart, hook setup,
detection reference, security model summary, residual risks), and add
`SECURITY.md` (SEC-16) and `CONTRIBUTING.md`, closing SEC-15 documentation
requirements.

## Context

Last issue: documents the product as built. README is the primary adoption
surface for an OSS security tool — accuracy about what the tool does *not*
protect against is part of the security posture.

## Scope

- `README.md` (replace the one-line stub), `SECURITY.md`,
  `CONTRIBUTING.md`. CHANGELOG bootstrap exists (26); RELEASING exists (26).

## Detailed Requirements

1. README sections, in order:
   - What it is + 30-second demo block (`printf` samples with expected
     output, all examples using escaped sequences — no raw hostile bytes in
     the README, same rule as the corpus).
   - Install: GitHub Releases binaries (per-target instructions + checksum
     verification command), `cargo install pastecheck`.
   - Quickstart: `pbpaste | pastecheck`, `--clipboard`, sanitize pipe.
   - Guard setup: zsh + bash `eval "$(pastecheck shell-init …)"` lines, the
     prompt key reference (p/a/s/d), `PASTECHECK_DISABLE`,
     `PASTECHECK_GATE_FAIL`, `PASTECHECK_BASH_MODE`.
   - What it detects: table derived from `pastecheck rules` output (26
     rules, category grouping, one-line examples).
   - Configuration: full `config.toml` reference (mirrors DESIGN §11).
   - **Limitations & residual risks (SEC-15, normative list)**: bracketed
     paste end-marker truncation; local hook does not guard remote shells
     over ssh (install remotely too); bash NUL fidelity; PC502 not flagged
     in non-ASCII-majority documents (ADR-002); pastecheck inspects, it
     does not make dangerous *commands* safe.
   - Exit codes table (§8.2) for scripting.
2. `SECURITY.md`: supported versions (latest minor), private reporting via
   GitHub Security Advisories (link to the repo's advisories page),
   response target (initial reply ≤ 7 days), scope note (the §3 threat
   model, incl. "reports about detection bypasses are in scope").
   Includes the repository-settings checklist for the owner: enable private
   vulnerability reporting, enable secret scanning + push protection
   (settings actions are performed by the human owner — handoff, ISSUE_PLAN
   §9).
3. `CONTRIBUTING.md`: build/test commands, corpus-case addition workflow
   (from 24's README), golden regeneration, lint gates, "adding a runtime
   dependency requires an ADR" rule (ADR-001), DCO-free simple policy.
4. Cross-checks: every command in the docs is copy-paste-runnable against
   the built binary; the rules table matches `pastecheck rules` output
   (manual diff during review; a doc-sync test is optional and out of
   scope).

## Acceptance Criteria

- [ ] Every shell block in README executes successfully against
      `target/release/pastecheck` (reviewer checklist in PR).
- [ ] README contains no raw control/bidi/invisible bytes
      (same grep as issue 24 applied to README/SECURITY/CONTRIBUTING).
- [ ] Residual-risks section lists all five SEC-15 items.
- [ ] SECURITY.md present with reporting channel + response target;
      owner settings checklist included.
- [ ] `pastecheck rules` row count (26) matches the README table.

## Validation

```sh
grep -rPc '[\x00-\x08\x0B-\x1F\x7F]' README.md SECURITY.md CONTRIBUTING.md  # all 0
# execute README code blocks manually per PR checklist
```

## Dependencies

Blocked by: 18, 20, 21, 22.

## Non-goals

Japanese localization (v2), man pages (v2), website.

## Design References

- `docs/DESIGN.md` §18, §3.3 SEC-15, SEC-16, §8.2, §11
- `docs/decisions/ADR-002-cjk-aware-homoglyph-policy.md` (limitations text)
