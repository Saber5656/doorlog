# 02 — Config, core types, SQLite store + migrations

## Summary
Load configuration (env > yaml > defaults), define the shared core
types (`event.Event`, `ingest.RawLine`, `ingest.Source`,
`parse.Parser`), and implement the SQLite store with migrations.

## Scope
- `internal/config`: full schema of DESIGN §13 with defaults,
  `DOORLOG_*` env overrides (nested keys via `_`), validation (paths
  absolute, times parseable, locale ∈ {ja,en}); `DOORLOG_NTFY_TOKEN`
  read from env only.
- `internal/event`: `Event` struct exactly as DESIGN §4; type +
  severity constants; ULID generation.
- `internal/ingest`: `RawLine{Path, Text, TS(time read)}` and
  `Source interface { Lines(ctx) <-chan RawLine }` (implementations in
  05/13/15).
- `internal/parse`: `Parser interface { Parse(RawLine) (RawEvent, bool) }`
  and `RawEvent` (pre-aggregation shape: kind, ts, user, ip, method,
  fingerprint, jail, action).
- `internal/store`: open (WAL, busy_timeout), embedded sequential
  migrations, `schema_migrations`; DDL of DESIGN §10; CRUD used by later
  issues: insert/update event, open-bucket lookup by (type,ip),
  cursor-paginated list, devices CRUD, settings get/set, digest
  upsert/list, notify-failure append/list, watermark get/set,
  retention prune. `events.backfilled` flag round-trips.
- Prepared statements only (T6).

## Acceptance criteria
- Fresh boot creates the schema; re-boot is idempotent; a v1→v2 dummy
  migration test proves the mechanism.
- Config precedence proven by tests (env beats yaml beats default).
- Store CRUD covered by unit tests against a temp DB, including
  cursor pagination ordering and the partial index path for open buckets.

## Validation
`go test ./internal/config/... ./internal/store/... ./internal/event/...`

## Dependencies
01. Blocks 03–13.

## Non-goals
Any parsing, aggregation, HTTP.

## Design references
DESIGN §4, §10, §13, §14 (T6).
