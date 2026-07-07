# Title

Input acquisition: sources, size cap, and UTF-8 policy (PC220)

## Summary

Implement `src/input.rs`: stdin/file source handling, the `max_bytes` hard
cap, the invalid-UTF-8 policy producing PC220, and line/column mapping,
per DESIGN §6 (SEC-4, SEC-5).

## Context

This is trust boundary B1. Every byte of input flows through this module;
its policy decisions (cap-then-refuse, lossy-decode-with-finding) are
security requirements, not conveniences.

## Scope

- `src/input.rs`; error variants in a new `src/error.rs` (`Usage`, `Io`,
  `SizeExceeded`, `ClipboardUnavailable`, `ConfigInvalid`) with exit-code
  mapping helper (`error.rs` is shared with issue 13).

## Detailed Requirements

1. `enum Source { Stdin, File(PathBuf), Clipboard }` — clipboard variant
   delegates to `clipboard::read()` (issue 19; stub returning
   `ClipboardUnavailable` until then).
2. `acquire(source, max_bytes) -> Result<Acquired, Error>`:
   - Reads at most `max_bytes + 1` bytes; if more than `max_bytes` were
     available, return `SizeExceeded { got_at_least, max }` **without**
     analyzing (DESIGN §6.2). Error text exactly:
     `input exceeds limit (<n>+ bytes > <max> bytes); not scanned`.
   - stdin: read from locked handle; FILE: `fs::File`; reject directories
     with `Io`.
3. UTF-8 policy: validate the byte buffer.
   - Valid ⇒ `Acquired { text: String, pc220: None, raw_bytes }`.
   - Invalid ⇒ collect maximal invalid runs as
     `(start_byte, end_byte, bytes ≤ 10 distinct)` in **original byte
     space**, build one PC220 seed (occurrences list) for the engine, decode
     with `String::from_utf8_lossy`, set `pc220: Some(...)`. Spans of the
     PC220 finding refer to original bytes; a `note` string
     ("offsets of other findings refer to lossy-decoded text") is set on the
     report per DESIGN §6.3.
4. Line map: precompute LF byte offsets over the analyzed text; expose
   `line_col(byte_offset) -> (u32, u32)` (1-based; column counts chars from
   line start). Used by all detectors via `ScanInput`.
5. Empty input is valid (0 bytes ⇒ clean).

## Acceptance Criteria

- [ ] Size cap: input of exactly `max_bytes` scans; `max_bytes + 1` returns
      `SizeExceeded` with the exact message format.
- [ ] Invalid UTF-8 (e.g. bytes `66 6f 6f 9b ff 62 61 72`) yields PC220 seed
      with correct byte spans and offenders `0x9B`, `0xFF`; lossy text is
      `foo��bar`.
- [ ] Lone-continuation, overlong, and truncated-multibyte sequences all
      count as invalid runs (table-driven test).
- [ ] `line_col` correct for: offset 0; offset after LF; multibyte chars
      before the offset (column counts chars, not bytes); CRLF input.
- [ ] Directory as FILE ⇒ `Io` error, exit-code mapping 3.

## Validation

```sh
cargo test --locked input:: error::
```

## Dependencies

Blocked by: 04.

## Non-goals

Clipboard implementation (19), CLI flag wiring (13).

## Design References

- `docs/DESIGN.md` §6, §3.3 SEC-4, SEC-5; §14 (error taxonomy)
