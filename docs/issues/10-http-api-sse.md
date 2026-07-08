# 10 — HTTP API + SSE stream

## Summary
Serve the JSON API and SSE stream per the DESIGN §11 contract table.

## Scope
- Every endpoint in DESIGN §11 with exact shapes; translated `text`
  attached to events/digests server-side using the active locale.
- `door_status` derivation for `/api/summary/today` (guarded / knocked /
  check; `check` iff unresolved alert in 24 h — resolved = a device now
  matches that event's user/ip/fingerprint).
- SSE `/api/stream`: `event.created`, `event.updated`,
  `summary.changed`; heartbeat comment every 25 s; last-event-id not
  required (UI refetches on reconnect).
- Origin check middleware on state-changing routes (T3); no cookies;
  settings PUT whitelist; ntfy token write-only.
- Contract tests: httptest against a seeded store — pagination cursor
  stability, severity filter, devices CRUD round-trip, settings
  whitelist rejection, Origin enforcement, SSE event delivery.

## Acceptance criteria
- All endpoints match the documented shapes (asserted via golden JSON
  where practical); nothing else registered.

## Validation
`go test ./internal/server/...`

## Dependencies
02, 06, 07. Blocks 11, 12.

## Non-goals
Any HTML rendering; auth (v1 non-goal, README covers it).

## Design references
DESIGN §11, §14 (T3).
