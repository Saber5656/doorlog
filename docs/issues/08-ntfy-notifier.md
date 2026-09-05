# 08 — ntfy notifier, delivery rules, retry

## Summary
Compose translated pushes and deliver via ntfy HTTP with the ADR-004
delivery classes and retry/backoff (DESIGN §9).

## Scope
- Delivery classification (immediate / timeline-only / digest) as a
  pure function of Event + config, unit-tested against the ADR-004
  table incl. `immediate_ban` toggle.
- ntfy client: POST with Title/Priority/Tags headers; bearer token from
  env; header-safe encoding of translated strings (T2);
  `include_ip: false` substitution in push payloads only.
- Retry 5 s / 30 s / 2 min, then record to `notify_failures` (store) with
  error string; store-first invariant: notify only after event commit.
- Rate floor: min 30 s between pushes; overflowed immediates degrade to
  digest-class with a `folded` marker (T5) — unknown-login alerts are
  exempt from folding.
- `POST /api/notify/test` handler logic (wired in Issue 10).
- Tests with httptest server: success, 5xx retry, auth header, rate
  floor, folding, include_ip.

## Acceptance criteria
- Rule table fully covered by tests; no goroutine leak under retry
  (leak detector in tests); failures visible via store.

## Validation
`go test ./internal/notify/...`

## Dependencies
07. Blocks 09.

## Non-goals
Apprise/other channels; digest content (09).

## Design references
DESIGN §9, §14 (T2/T5), ADR-004, ADR-005.
