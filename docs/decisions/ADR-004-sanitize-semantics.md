# ADR-004: Sanitize is opt-in, stdout-only, and never guesses intent

Status: accepted (2026-07-08, confirmed with product owner: "report +
interactive confirm + optional sanitize")

## Context

A tool that silently rewrites text trades one integrity problem for another:
mis-sanitization corrupts legitimate content (emoji, Japanese IVS, multi-line
scripts), destroys user trust, and is hard to debug. But detection without a
remedy leaves users hand-editing invisible characters, which is impractical.

## Decision

1. Sanitization never happens implicitly. `check` and `gate` never modify;
   `sanitize` (or the hook's `s` key) is an explicit act.
2. Output goes to **stdout only** — never in-place, never to the clipboard
   (SEC-6; pastecheck must not become the clipboard-writing tool it warns
   about).
3. Per-category strategies with conservative defaults (DESIGN §10):
   controls/invisibles/bidi-overrides strip; newlines are **kept** (changing
   line structure changes script semantics); homoglyph words are **never**
   auto-rewritten — only the deterministic fullwidth/punctuation tables map
   to ASCII, and only under opt-in `ascii-map`.
4. Legitimate joiners are preserved: emoji ZWJ/VS16 sequences and Han+IVS
   (PC304 context) survive byte-exact.
5. Every run prints a per-category stderr summary including what was *not*
   fixed, so "sanitized" is never mistaken for "clean" (abuse case §3.4).
6. Guarantees are property-tested: idempotent; effective for strip/map
   categories; introduces no codepoints beyond the documented ASCII targets.

## Alternatives considered

- **Auto-sanitize by default in the hook**: smoothest UX, rejected — silent
  content modification is the exact trust failure this tool exists to
  prevent, and rewriting without display would hide attacks instead of
  surfacing them.
- **Skeleton-based homoglyph auto-correction** (rewrite `pаypal`→`paypal`):
  plausible-looking but unsound — the "correct" word cannot be inferred
  (the confusable might be the intended text). Rejected permanently, not
  just for v1.
- **In-place file sanitize / clipboard write-back**: violates SEC-6 and
  least-surprise; deferred (write-back listed as a v2 discussion item).

## Consequences

- The hook's `s` path is safe to press without reading the diff.
- Users needing aggressive normalization opt in per category via config.
- Sanitized-but-still-flagged content (PC501 words) remains visible in the
  summary — by design.
