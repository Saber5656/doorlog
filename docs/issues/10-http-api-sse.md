# 10 — HTTP API + SSE stream

## Summary
Serve the JSON API and SSE stream per the DESIGN §11 contract table.

## Scope
- Every endpoint in DESIGN §11 with the normative wire shapes of
  DESIGN §11.1 (golden JSON tests); translated `text` attached to
  events/digests server-side using the active locale.
- `door_status` derivation for `/api/summary/today` (guarded / knocked /
  check; `check` iff unresolved alert in 24 h — resolved = a device now
  matches that event's user/ip/fingerprint).
- SSE `/api/stream`: `event.created`, `event.updated`,
  `summary.changed`; heartbeat comment every 25 s; last-event-id not
  required (UI refetches on reconnect).
- Guard middleware (T3): the DESIGN §11 built-in Host rule (IP
  literals, localhost, single-label, .local/.home.arpa/.lan/.internal)
  plus `trusted_hosts` on all routes, Origin check on state-changing
  routes; no cookies.
- Settings PUT whitelist per §11 (no URLs, no token — T7); GET returns
  the ntfy destination masked.
- Contract tests: httptest against a seeded store — pagination cursor
  stability, severity filter, devices CRUD round-trip incl. 422
  validation, settings whitelist rejection (attempt to set ntfy_url →
  422), Host/Origin enforcement (403), SSE event delivery.

## Acceptance criteria
- All endpoints match the documented shapes (asserted via golden JSON
  where practical); nothing else registered.

## Validation
`go test ./internal/server/...`

## Dependencies
02, 06, 07, 08 (`/api/notify/test` + failures list need the notifier).
Blocks 11, 12, 13.

## Non-goals
Any HTML rendering; auth (v1 non-goal, README covers it).

## Design references
DESIGN §11, §14 (T3).
