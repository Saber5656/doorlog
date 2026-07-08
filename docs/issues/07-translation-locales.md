# 07 — Translation layer + ja/en locales

## Summary
Implement the locale-file renderer and author the full `ja.yaml` and
`en.yaml` (DESIGN §8) — the product's voice lives in this issue.

## Scope
- Loader: embedded locales, optional override dir (`/data/locales/`);
  strict render (missing key or missing placeholder value = error).
- Key scheme `<type>.<variant>.<context>`; variant resolution rules
  (known/unknown; one_private/one/few/many with 1 / 2–9 / 10+);
  context = timeline / push / push_title; plus `digest.*`,
  `door_status.*`, `geo.*`, `countries.*`, `ui.*`.
- Author both locales completely, following DESIGN §8 starting content
  and the four tone rules; en is a real translation, not literal.
- Golden tests: every (type × variant × context × locale) rendered
  against `testdata/golden/*.txt`; CI fails on key-set mismatch between
  ja and en; tone lint: golden files grepped for banned jargon tokens
  ("SSH " bare, "brute force", "認証失敗") outside parentheses.
- `GET /api/locale` payload shape prepared as a marshalable struct
  containing exactly the `ui:` + `door_status:` namespaces (served in
  Issue 10; all other namespaces are server-render-only per §11).

## Acceptance criteria
- ja/en key sets identical; goldens pass; strict-mode error paths
  tested; README-facing sample sentences match DESIGN §1 table.

## Validation
`go test ./internal/translate/...`

## Dependencies
02. Blocks 08, 09, 10.

## Non-goals
LLM anything; locales beyond ja/en (structure must allow them).

## Design references
DESIGN §8, ADR-003.
