# Research: Terminal Paste Threat Catalog

Status: reviewed input for `docs/DESIGN.md` §3 (security model) and §7 (detector specs).
Date: 2026-07-08.

This document catalogs the attack classes that make pasting text into a terminal
dangerous. Each class maps to a pastecheck detector category. The catalog is the
basis for the rule registry in `docs/DESIGN.md` §5.3.

## Attack classes

### T1. Immediate execution via embedded line breaks

Pasted text containing `LF` or `CR` is executed line-by-line by the shell the
moment it is pasted, unless the terminal/shell pair uses bracketed paste mode.
A lone `CR` (U+000D) is treated as Enter by terminals in canonical mode, so it
triggers execution even when no `LF` is present. A single **trailing** newline
is the classic form: the victim believes they will get a chance to review the
command, but it runs immediately.

Copy-time hijacking ("pastejacking") makes this worse: JavaScript `copy` event
handlers or CSS-hidden text cause the clipboard to contain different content
than what the victim visually selected, including appended newlines and extra
commands. Reference PoC: dxa4481/Pastejacking (https://github.com/dxa4481/Pastejacking).

Related lookalikes: `NEL` (U+0085), `LS` (U+2028), `PS` (U+2029), `VT` (U+000B),
`FF` (U+000C) are rendered as line breaks by many viewers (so they *look* like
formatting) but are **not** submitted as Enter by shells — a visual/semantic
mismatch in the other direction that hides content structure.

Detector mapping: `newline` (PC101–PC105).

### T2. Terminal control-sequence injection

Text containing `ESC` (U+001B) or C1 controls (U+0080–U+009F, e.g. 0x9B = CSI)
is interpreted by the terminal emulator when echoed. Consequences include:

- Hiding or rewriting displayed content (cursor movement, erase-line/screen,
  SGR "conceal", overwriting with `BS`/mid-line `CR`) so a reviewed command is
  not the executed command.
- Window/tab title injection via `OSC 0/2` (spoofing context, and historically
  title-report reflection attacks).
- **Clipboard overwrite via `OSC 52`**: pasted text can silently replace the
  clipboard, so the *next* paste is attacker-controlled.
- Terminal-emulator parser bugs: `DCS`/`SOS`/`PM`/`APC` strings and
  malformed sequences have repeatedly caused RCE-class CVEs in emulators
  (e.g. iTerm2 CVE-2019-9535, CVE-2024-38396).

Even "harmless" sequences are hostile in paste context because the user cannot
see them. Any raw `ESC`/C1 in a paste is a finding.

Detector mapping: `control` (PC201–PC212, PC220).

### T3. Invisible and zero-width characters

Zero-width and format characters (`ZWSP` U+200B, `ZWNJ` U+200C, `ZWJ` U+200D,
`WORD JOINER` U+2060, `ZWNBSP/BOM` U+FEFF, `SOFT HYPHEN` U+00AD, invisible
math operators U+2061–U+2064, Hangul fillers, etc.) change the semantics of
commands, URLs, hostnames, identifiers, and config values while being visually
undetectable. Typical impacts:

- `curl https://examp<U+200B>le.com` (ZWSP inside the hostname) resolves to a
  different or non-existent domain.
- A BOM prefix (`<U+FEFF>ls`) makes the command unknown; in scripts it breaks
  shebang processing.
- Invisible characters inside pasted source code survive review and alter
  string comparisons or identifiers.

Special case — **Unicode tag characters** (U+E0000–U+E007F): a parallel,
completely invisible ASCII alphabet. Used for "ASCII smuggling": hiding
payloads/instructions inside innocuous-looking text. This is also the primary
carrier for hidden LLM prompt-injection in copied text, which matters because
terminal users increasingly paste into AI CLIs. Variation selectors
(U+FE00–U+FE0F, U+E0100–U+E01EF) are likewise abused as a data-smuggling
channel when they appear in long runs or detached from legitimate bases.

False-positive constraint: `ZWJ` and `VS16` are *legitimate* inside emoji
sequences (the family emoji is three pictographs joined by ZWJ: 👨 U+200D 👩 U+200D 👧), and ideographic variation
sequences (IVS, U+E0100–) after Han ideographs are legitimate in Japanese
text. The detector must be emoji- and IVS-aware (DESIGN §7.3).

Detector mapping: `invisible` (PC301–PC304).

### T4. Bidirectional-override reordering (Trojan Source)

Bidi control characters (`LRE/RLE/LRO/RLO/PDF` U+202A–U+202E and isolates
`LRI/RLI/FSI/PDI` U+2066–U+2069) reorder what is *displayed* without changing
the byte order that is *executed/compiled*. The Trojan Source paper
(CVE-2021-42574, https://trojansource.codes/) demonstrated review-proof
malicious source code; the same technique makes a pasted shell command read
benign while executing something else. GitHub, compilers (rustc 1.56.1), and
code review tools added warnings for these characters in response — a paste
guard is the equivalent protection at the terminal boundary.

Directional *marks* (`LRM` U+200E, `RLM` U+200F, `ALM` U+061C) are weaker
(no reordering of strong-directional runs) and common in legitimate RTL text,
so they warrant a lower default severity.

Detector mapping: `bidi` (PC401–PC403).

### T5. Homoglyph / confusable substitution

Characters that render (near-)identically to ASCII substitute a different
target for the one the user believes they see:

- Cross-script letter swap: Cyrillic `а` (U+0430) in `pаypal.com`
  (CVE-2021-42694 is the "homoglyph in source" variant of Trojan Source).
- Whole-word substitution: `раураl` entirely in Cyrillic.
- Fullwidth forms: `ｒｍ` (U+FF52 U+FF4D) — also a common accident when
  copying from CJK documents; the command silently fails or hits the wrong
  path.
- Lookalike punctuation from rich-text sources: smart quotes U+2018/U+2019/
  U+201C/U+201D, en/em dashes U+2013/U+2014, minus sign U+2212, fraction
  slash U+2044. These break or alter commands (`--flag` vs `—flag`,
  `"quoted"` vs `“quoted”`) and are the single most frequent real-world
  paste corruption from blogs/chat/word processors.

Unicode TR39 (https://www.unicode.org/reports/tr39/) defines the standard
machinery: confusable "skeleton" mapping and mixed-script detection. The
critical false-positive constraint for pastecheck: **CJK text is normal
terminal input for our primary users.** Pure Japanese/Chinese/Korean tokens
must never be flagged as homoglyph attacks; only confusable-with-ASCII
substitution should alarm (DESIGN §7.5 and ADR-002).

Detector mapping: `homoglyph` (PC501–PC504).

### T6. Copy-time content substitution (context, not a detector)

CSS/JS clipboard hijacking (T1 reference) means "what you selected" and "what
you copied" routinely differ. pastecheck cannot prevent the substitution; its
value is that every substitution technique above becomes *visible at paste
time*, which is the last boundary before execution. This motivates the shell
hook (guard) form factor rather than inspect-on-demand only.

### T7. Denial / annoyance inputs

Adversarial inputs aimed at the guard itself: multi-megabyte pastes, inputs
with hundreds of thousands of findings (e.g. 1 MiB of ESC bytes), invalid
UTF-8, and pathological sequences designed to hang naive regex-based scanners.
The guard must have hard input-size limits, linear-time detectors, bounded
report output, and a no-regex parsing policy for untrusted input
(DESIGN §3.3 SEC requirements, §15).

## Out of scope for v1 (recorded for v2 triage)

- Content-level dangerous-command heuristics (`curl | sh`, `rm -rf` patterns)
  — high false-positive rate, different product axis (semantic vs lexical).
- Suspicious visible whitespace (NBSP U+00A0, thin spaces) — deferred by
  scope decision 2026-07-08 (see ISSUE_PLAN "Deferred").
- URL/punycode-specific analysis, IDN policy.
- Private-use (Co) and unassigned (Cn) codepoint flagging.
- Clipboard *write-time* monitoring (daemon form factor).

## References

- Trojan Source: Invisible Vulnerabilities (Boucher & Anderson) — https://trojansource.codes/
- CVE-2021-42574 (bidi), CVE-2021-42694 (homoglyph)
- Pastejacking PoC — https://github.com/dxa4481/Pastejacking
- Unicode TR39 Security Mechanisms — https://www.unicode.org/reports/tr39/
- Unicode TR36 Security Considerations — https://www.unicode.org/reports/tr36/
- iTerm2 CVE-2019-9535 (tmux integration RCE), CVE-2024-38396 (window-title escape RCE)
- OSC 52 clipboard access — xterm ctlseqs documentation (https://invisible-island.net/xterm/ctlseqs/ctlseqs.html)
- GitHub bidi warning rollout — https://github.blog/changelog/2021-10-31-warning-about-bidirectional-unicode-text/
