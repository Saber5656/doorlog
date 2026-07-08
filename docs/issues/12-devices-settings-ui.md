# 12 — Devices & settings UI

## Summary
Build `#/devices` and `#/settings` (DESIGN §12.2) against the Issue 10
API.

## Scope
- Devices: list (emoji, name, matchers summary), add/edit/delete modal
  (emoji picker = curated 12-emoji grid, free text fallback); accepts
  pre-fill query params from timeline alert cards and highlights the
  triggering event context.
- Settings: locale switch (ja/en, live reload of `/api/locale`), digest
  time picker + skip-empty toggle, immediate-ban toggle, include-IP
  toggle, ntfy destination shown read-only/masked with a "change it in
  doorlog.yml" hint (DESIGN T7 — no URL/token fields exist), test-
  notification button with inline result, delivery failures list.
- Optimistic UI with server-error rollback; all strings from locale.

## Acceptance criteria
- Vitest component tests: device CRUD round-trip against mocked API,
  settings whitelist behavior, masked destination rendering (no
  unmasked URL in DOM); axe-core pass.

## Validation
`cd web && npm run check && npm test`

## Dependencies
10. Blocks 14.

## Non-goals
Auth/user management; per-device notification rules (v2).

## Design references
DESIGN §12.2, §11.
