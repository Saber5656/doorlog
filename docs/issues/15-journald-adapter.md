# 15 — journald source adapter (optional for v1)

## Summary
Second `Source` implementation reading sshd entries from journald, for
hosts without rsyslog (ADR-006, U5). Optional for the v1 release.

## Scope
- Exec `journalctl -t sshd -f -o json` (also `-t sshd-session`),
  parse JSON lines, adapt `__REALTIME_TIMESTAMP` + `MESSAGE` into
  `RawLine` feeding the existing sshd parser (message-only regexes).
- Cursor persistence via `--after-cursor` + stored cursor (replaces
  inode watermark for this source).
- Config: `ingest.journald: true` switches sshd ingest (fail2ban stays
  file-based); process restart/backoff supervision.
- Docs: required mounts (`/var/log/journal:ro`, `/etc/machine-id:ro`)
  and image variant note if journalctl adds meaningful size.
- Tests: fixture JSON lines through the adapter; supervision restart
  test with a fake command.

## Acceptance criteria
- On a journald-only host, login events flow end-to-end; restart
  resumes from cursor without duplicates.

## Validation
`go test ./internal/ingest/... -run TestJournald` + manual host check.

## Dependencies
05 (Source conventions, watermark plumbing).

## Non-goals
Reading fail2ban from journald; sd-journal C bindings.

## Design references
DESIGN §15, §18 (U5), ADR-006.
