# 06 — Aggregator (failed-attempt buckets) + enrichment

## Summary
Fold raw failed attempts into per-IP bucket events, and enrich all
events with known-device matching and geo (DESIGN §6, §7).

## Scope
- Bucket lifecycle exactly as DESIGN §6: open on first failed attempt;
  in-place update (count, window_end) with SSE `event.updated`
  emission hook (callback interface; SSE itself is Issue 10); close on
  15 min age / 30 min idle / digest fire, persisting
  `bucket_closed_at` + `bucket_close_reason` (closed buckets never
  reopen); per-(ip,second) dedup.
- Severity rules: notice; count=1 ∧ private → ok.
- login.success path: device match (fingerprint first, then user+ip,
  ip may be CIDR) → DeviceID + severity ok; no match → alert
  (immediate-class).
- Geo: `Private` per DESIGN §7.2 (incl. CGNAT→tailnet wording flag);
  optional mmdb lookup behind an interface (nil-safe when unconfigured).
- All time logic against an injected clock; fake-clock unit tests:
  open→increment→idle-close, age-close, digest force-close, dedup,
  two parallel IPs, backfilled lines never trigger immediate-class.

## Acceptance criteria
- Bucket tests green incl. restart safety (open bucket found via store
  partial index and continued); device matching precedence tested;
  private/CGNAT/public classification table-tested.

## Validation
`go test ./internal/aggregate/... ./internal/enrich/...`

## Dependencies
02, 03, 04 (fail2ban RawEvents map to events here). Blocks 09, 10, 13.

## Non-goals
Translation, pushing, digest math (09).

## Design references
DESIGN §4.1, §6, §7, ADR-004.
