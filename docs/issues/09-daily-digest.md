# 09 — Daily digest: stats rollup, scheduler, push

## Summary
Compute the day's stats, store the digest, render per-locale, and push
at the configured time (DESIGN §10 digests, §8 digest keys).

## Scope
- Scheduler: fire at `digest.time` local (config TZ), catch-up rule —
  on boot, if last digest date < today and now > digest time, fire once
  (no double-send: `digests.date` UNIQUE + `sent_at`).
- Rollup since previous digest date boundary (00:00 local), excluding
  `backfilled=1` events (ADR-006): `ip_count`, `attempt_count` (sum of
  bucket counts), `banned_count`, `login_count`, `top_ips` (≤3, with
  country) → `stats_json`.
- Force-close open buckets at digest time (via aggregator hook).
- Zero-activity day renders `digest.none` (still stored, still pushed —
  the calm heartbeat is a feature; config `digest.skip_empty: false`
  default documented in configuration.md).
- Render on read for UI; render at send time for push locale.
- Fake-clock tests: normal fire, boot catch-up, no double-send, empty
  day, stats correctness against seeded store.

## Acceptance criteria
- Digest math proven against a seeded day incl. bucket force-close;
  UNIQUE constraint prevents double-send under restart loops.

## Validation
`go test ./internal/digest/...`

## Dependencies
06, 07, 08. Blocks 14.

## Non-goals
Weekly/monthly digests; LLM prose.

## Design references
DESIGN §8 (digest keys), §9, §10 (digests table), ADR-004.
