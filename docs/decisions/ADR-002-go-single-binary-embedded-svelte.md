# ADR-002: Go single binary with embedded Svelte UI, SQLite storage

Date: 2026-07-08
Status: accepted (user-approved, 2026-07-08)

## Context

Target deployment is a family's home server (user's dogfooding host:
Linux + systemd; Raspberry Pi class hardware must stay comfortable).
Distribution friction and idle resource usage matter more than raw
development speed. Candidates considered: Go + Svelte, TypeScript
monorepo (Hono + Svelte/React, Uptime Kuma path), Python (FastAPI + HTMX).

## Decision

- Backend: Go (>= 1.24), one static binary; Svelte 5 + Vite frontend
  built to static assets and embedded via `go:embed`.
- Storage: SQLite via `modernc.org/sqlite` (pure Go, no CGO) so
  cross-compilation and distroless/alpine images stay trivial.
- No runtime Node dependency; Node 22 is a build-time dependency only.

## Consequences

- Tiny image, low idle memory, single-file binary release path
  (goreleaser later) — good fit for Pi-class hosts and homelab tastes.
- Two languages in the repo; contributors need Go and Svelte. Accepted:
  the UI surface is small (one timeline page + two minor pages).
- SQLite chosen over flat JSON for aggregation queries (digest math,
  cursor pagination) and retention pruning.
