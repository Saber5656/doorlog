# 04 — fail2ban parser

## Summary
Parse fail2ban action lines (Ban / Unban / Restore Ban) into
`RawEvent`s per DESIGN §5.2.

## Scope
- Regex of DESIGN §5.2; keep jail name in `RawEvent`; preserve
  `action` verbatim (`Ban` / `Unban` / `Restore Ban`) — the event
  mapping in Issue 06 turns `Restore Ban` into `fail2ban.banned` with
  `Event.Variant="restore"` for the gentler template.
- Same robustness rules as Issue 03 (§5.3).
- Fixtures `testdata/fail2ban/`: Ban, Unban, Restore Ban, IPv6 ban,
  non-NOTICE noise ignored, WARNING lines ignored, oversized line.

## Acceptance criteria
- Table-driven fixture test asserts full `RawEvent` equality; ignored
  lines return `false`.

## Validation
`go test ./internal/parse/... -run TestFail2ban`

## Dependencies
02. Blocks 06, 13.

## Non-goals
Jail-specific translation (v1 assumes sshd jail); fail2ban socket API.

## Design references
DESIGN §5.2, §5.3, §18 (U2).
