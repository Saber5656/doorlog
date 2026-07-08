# 14 — README, GIF, 5-minute setup & privacy guide

## Summary
Author the public face: README with the demo GIF, copy-paste setup,
and honest privacy/security notes (DESIGN §1 success criteria, §17).

## Scope
- README.md (en primary, `README.ja.md` full ja mirror): hero GIF
  (recorded from demo mode: unknown login alert → knock burst → digest),
  one-paragraph pitch, ja/en sample translations table (DESIGN §1),
  5-minute setup (compose block from §16 verbatim), ntfy app pointer,
  known-devices onboarding, configuration reference link.
- `docs/configuration.md` generated-from/verified-against the §13
  default yaml (drift check in CI: parse the README compose + config
  doc against the config schema).
- Privacy/security section: LAN-bind + reverse-proxy note (no auth in
  v1), ntfy.sh topic-as-capability + self-host recommendation,
  include_ip option, "your logs never leave the host except pushes".
- rsyslog note for journald-only hosts (ADR-006); Fedora `/var/log/secure`
  known-limitation note (U1); GeoLite2 attribution wording (U3).
- Repo hygiene: LICENSE (MIT), NOTICE (fonts OFL), CONTRIBUTING stub,
  issue templates pointing at docs/.

## Acceptance criteria
- A fresh reader can go compose-copy → running UI in ≤5 min on a stock
  Ubuntu host (dogfooding host walkthrough recorded as evidence);
  GIF ≤ 6 MB, loops cleanly; config drift check green.

## Validation
Manual walkthrough on the dogfooding host + CI drift check.

## Dependencies
09, 11, 12, 13.

## Non-goals
Docs site; blog/launch posts.

## Design references
DESIGN §1, §13, §14 (privacy), §16, §17, §18 (U1/U3).
