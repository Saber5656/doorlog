# 11 — Timeline UI

## Summary
Build the main Svelte page: "today at your door" header card + event
feed with the knock animation (DESIGN §12.1, §12.3).

## Scope
- Header card: door SVG (three states wired to `door_status`), status
  line + today's numbers from `/api/summary/today`.
- Feed: date separators, event cards (severity-tinted icon circle,
  translated sentence, relative time), digest cards interleaved by
  date, upward infinite scroll on the events cursor.
- Alert cards: "name this device" button pre-filling the device form
  (navigates to `#/devices` modal with query params; POST wired in 12 —
  behind a shared store so 11 is testable standalone with a stub).
- SSE wiring: `event.created` prepends + door knock wiggle (CSS
  keyframes ≤0.6 s, `prefers-reduced-motion` respected);
  `event.updated` updates bucket counts in place; reconnect → refetch.
- Visual identity tokens as CSS custom properties (DESIGN §12.3 hex
  values, radii, fonts); light/dark via `prefers-color-scheme` +
  manual toggle persisted in localStorage.
- Fonts: subset M PLUS Rounded 1c + Nunito woff2 bundled (U4: total
  < 300 KB or fall back to system rounded stack); zero external
  requests (CI asserts no http(s) URLs in built CSS/JS beyond same-origin).
- All strings from `/api/locale`; no hardcoded user-facing text.
- Log-derived values rendered as text nodes only; `{@html}` absent (T1).

## Acceptance criteria
- `npm run check` clean; component tests (Vitest) for feed rendering
  from fixture JSON, SSE prepend, reduced-motion path; axe-core pass on
  the page (no serious violations); Lighthouse perf ≥ 90 on demo data.

## Validation
`cd web && npm run check && npm test && npm run build`

## Dependencies
10. Blocks 14.

## Non-goals
Devices/settings pages (12); PWA/push.

## Design references
DESIGN §12.1, §12.3, §14 (T1), §18 (U4).
