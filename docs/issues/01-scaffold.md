# 01 — Scaffold: repo layout, build pipeline, CI, healthz

## Summary
Create the buildable skeleton: Go module + Svelte app + embed wiring +
Dockerfile + compose + CI, serving `/healthz` and the (placeholder) UI.

## Scope
- Repo layout exactly as DESIGN §3.1 (empty packages may contain only
  doc.go); `cmd/doorlog/main.go` with flag/env bootstrapping and
  graceful shutdown (SIGTERM drains HTTP, closes store).
- `web/`: Svelte 5 + Vite + TypeScript, `npm run build` emitting into
  `internal/server/dist/`; `go:embed` serves it at `/`.
- `GET /healthz` → `200 {"version":"<git describe>","demo":false}`.
- Multi-stage Dockerfile (DESIGN §16), non-root, image < 40 MB;
  `docker-compose.yml` as in DESIGN §16.
- CI workflow: vet, golangci-lint, banned-pattern grep (`{@html}`),
  go test, svelte-check, web build, docker build, demo smoke step
  stubbed (activated in Issue 13).
- `Makefile` (or `Taskfile`): `make dev` (concurrently vite dev + go
  run), `make build`, `make test`.

## Acceptance criteria
- `make build` produces one binary that serves UI + healthz with no
  Node at runtime.
- `docker build` succeeds; `docker run -p 8090:8090 <img>` → healthz 200.
- CI green on the PR; image size gate in CI: fail > 60 MB, warn > 40 MB
  (DESIGN §16).

## Validation
`make build && ./doorlog & curl -fsS localhost:8090/healthz`
`docker build -t doorlog:dev . && docker run --rm -d -p 8090:8090 doorlog:dev && curl -fsS localhost:8090/healthz`

## Dependencies
None. Blocks all other issues.

## Non-goals
Any real pipeline logic, storage, or UI content.

## Design references
DESIGN §3.1, §16, §17.
