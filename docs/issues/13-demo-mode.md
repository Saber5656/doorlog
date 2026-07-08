# 13 — Demo mode

## Summary
Bundle a 24 h scenario and replay it accelerated so the full product
can be experienced (and GIF-recorded) with zero setup (DESIGN §15).

## Scope
- `testdata/demo/`: authored scenario with relative-offset timestamps
  covering: two known-device logins (devices seeded), one
  unknown-device login, three bot bursts from distinct IPs (counts 12 /
  47 / 150 → exercises few/many variants), one ban + restore-ban, one
  private-IP single typo, quiet stretches.
- Replayer implements `Source`: maps offsets onto now, ~40× speed,
  loops; seeds demo devices on first boot; `source: demo` on events.
- `DOORLOG_DEMO=1` or `demo: true` activates it, disables file tailers
  and real pushes (notifier goes to a log sink + UI), shows the UI demo
  ribbon.
- Digest fires "daily" in compressed time so the digest card appears
  within a loop.
- CI smoke (activates the Issue 01 stub): run image with demo env, wait
  healthz, assert ≥1 event of each class via `/api/events`, assert
  summary parses.

## Acceptance criteria
- One loop shows every template variant at least once (assertion via
  API in an integration test); no ntfy egress in demo (httptest guard).

## Validation
`go test ./internal/ingest/... -run TestDemo && DOORLOG_DEMO=1 go run ./cmd/doorlog & curl localhost:8090/api/events`

## Dependencies
03, 04, 05, 06. Blocks 14.

## Non-goals
Synthetic load testing; configurable scenarios.

## Design references
DESIGN §15, §17 (CI smoke).
