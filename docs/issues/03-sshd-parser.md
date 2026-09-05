# 03 — sshd auth.log parser

## Summary
Parse OpenSSH sshd lines (auth.log format) into `RawEvent`s per
DESIGN §5.1, with fixtures and a fuzz seed corpus.

## Scope
- Syslog prefix handling: RFC3164 (year completion incl. Dec→Jan rule,
  host TZ) and RFC3339/RSYSLOG_FileFormat; process name
  `sshd(-session)?` (U1).
- Message patterns: Accepted (with optional keytype+fingerprint),
  Failed (incl. `invalid user`), Invalid user, Connection closed
  `[preauth]` — mapped exactly as the DESIGN §5.1 table.
- Robustness: 8 KB truncation before regex; strip C0/C1 + newlines from
  usernames; 64-rune cap (T2, T4); unrecognized lines → `false` +
  ignored-counter log.
- Fixtures `testdata/authlog/`: ≥14 lines as enumerated in DESIGN §5.1
  (both timestamp styles, IPv6, UTF-8/space username, oversized line,
  ignored lines). Table-driven test asserts full `RawEvent` equality.
- `FuzzParse` with fixtures as seed corpus; runs 30 s in nightly CI,
  seed-only in PR CI.

## Acceptance criteria
- All fixtures pass; year completion tested around Dec 31/Jan 1;
  fuzz finds no panics on seed corpus run.

## Validation
`go test ./internal/parse/... -run TestSSHD`
`go test ./internal/parse/... -fuzz FuzzParseSSHD -fuzztime 10s`

## Dependencies
02. Blocks 06, 13.

## Non-goals
Aggregation, fail2ban lines, journald JSON shape.

## Design references
DESIGN §5.1, §5.3, §14 (T2/T4), §18 (U1).
