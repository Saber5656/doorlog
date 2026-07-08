# 05 — File tailer with rotation and watermark persistence

## Summary
Implement the polling file `Source` (DESIGN §15, ADR-006): follow a
file, survive rotation/truncation, persist (inode, offset) watermarks.

## Scope
- 1 s polling (configurable); emit `RawLine` per line; partial last
  line buffered until newline.
- Rotation: inode change → finish old file, reopen new at offset 0.
  Truncation: size < offset → reset to 0. Missing file: retry with
  30 s log throttle (fail2ban may not exist → path "" disables).
- Watermark `{inode, offset}` saved via store after each batch;
  on start: resume if inode matches, else start at EOF (or offset 0
  when `ingest.backfill: true`).
- Backfill mode: read existing content once; events older than the
  digest watermark are timeline-only (no push, no digest counting) —
  coordinate via a `Backfilled bool` on `RawLine`.
- Integration-style tests with temp files: append, rotate (rename+new),
  truncate, restart-resume, backfill flag propagation.

## Acceptance criteria
- No line lost or duplicated across rotation and restart in tests;
  EOF-start default proven (pre-existing lines not emitted unless
  backfill).

## Validation
`go test ./internal/ingest/... -run TestTailer`

## Dependencies
02. Blocks 13, 15.

## Non-goals
journald (15), demo replay (13), parsing.

## Design references
DESIGN §15, §10 (watermarks), ADR-006.
