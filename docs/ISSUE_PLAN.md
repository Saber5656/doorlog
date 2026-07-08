# doorlog v1 issue plan

Source of truth for scope/behavior: `docs/DESIGN.md` (linked per issue).
15 issues, 3 waves. 1 issue = 1 branch/worktree = 1 PR. Issue 15 is
optional for the v1 release.

## Waves

| Wave | Theme | Issues |
|---|---|---|
| 1 — foundation | build/run skeleton, storage, parsing, tailing | 01–05 |
| 2 — brain | aggregation, enrichment, translation, notify, digest | 06–09 |
| 3 — face | API, UI, demo, README (+ optional journald) | 10–15 |

## Issue list

| # | Title | Depends on | Blocks |
|---|---|---|---|
| 01 | Scaffold: repo layout, build pipeline, CI, healthz | — | all |
| 02 | Config, core types (Event/RawLine/interfaces), SQLite store + migrations | 01 | 03–13 |
| 03 | sshd auth.log parser (fixtures, fuzz seed) | 02 | 06, 13 |
| 04 | fail2ban parser (fixtures) | 02 | 06, 13 |
| 05 | File tailer (rotation, truncation, watermark persistence) | 02 | 13, 15 |
| 06 | Aggregator (failed-attempt buckets) + enrichment (devices, geo) | 02, 03 | 09, 10, 13 |
| 07 | Translation layer + locales ja/en (golden tests) | 02 | 08, 09, 10 |
| 08 | ntfy notifier + delivery rules + retry | 07 | 09 |
| 09 | Daily digest (stats rollup, scheduler, push) | 06, 07, 08 | 14 |
| 10 | HTTP API + SSE stream | 02, 06, 07 | 11, 12 |
| 11 | Timeline UI (header card, feed, knock animation) | 10 | 14 |
| 12 | Devices & settings UI | 10 | 14 |
| 13 | Demo mode (bundled 24 h scenario, accelerated replay) | 03, 04, 05, 06 | 14 |
| 14 | README, GIF, 5-minute setup & privacy guide | 09, 11, 12, 13 | — |
| 15 | journald source adapter (optional for v1) | 05 | — |

Suggested serial order for a single implementer:
01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11 → 12 → 13 → 14 (→ 15).
Parallelizable pairs: (03,04,05), (07 with 06), (11,12), (13 with 11/12).

## Cross-cutting rules (apply to every issue)

- Definition of Done = its Acceptance criteria + its Validation commands
  green + CI green + no `{@html}` / no new lint suppressions.
- Security-relevant acceptance lines (DESIGN §14 T1–T6) are not
  skippable; if an issue touches a listed threat, the mitigation test
  ships in the same PR.
- All user-facing strings go through the locale files — hardcoded UI
  text fails review.
- Every PR updates `docs/` if behavior diverges from DESIGN (DESIGN is
  amended in the same PR, never silently contradicted).

## v1 release checklist

Everything in DESIGN §17 "v1 release acceptance", verified on the
dogfooding host (Ubuntu-family, systemd, sshd + fail2ban), plus:
GHCR image published from a tag; Vault task record updated with the
release evidence.
