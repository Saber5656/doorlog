# ADR-006: file tailing is the primary v1 input; journald adapter is optional

Date: 2026-07-08
Status: accepted

## Context

Debian/Ubuntu-style hosts write sshd auth events to `/var/log/auth.log`
(rsyslog) and fail2ban to `/var/log/fail2ban.log` — both trivially
readable from a container via read-only bind mounts. journald-only hosts
require `journalctl` inside the image plus `/var/log/journal` +
`/etc/machine-id` mounts, which complicates the "compose up and done"
story. The dogfooding host is Linux/systemd where rsyslog is available.

## Decision

- Primary ingest: polling file tailer (1 s interval) on bind-mounted
  files; handles rotation via inode change and truncation via size
  regression; persists (inode, offset) so restarts do not re-notify.
- On start, begin at end-of-file by default (`ingest.backfill: false`);
  optional backfill reads the existing file once for timeline seeding.
  Backfilled events are marked (`events.backfilled=1`) and permanently
  excluded from pushes and digest counts — timeline display only.
- journald adapter (exec `journalctl -f -o json`) is designed as a second
  implementation of the same `Source` interface, scheduled as a
  follow-up issue (wave 3, optional for v1 release).
- README documents the rsyslog fallback for journald-only hosts.

## Consequences

- v1 works with two `:ro` bind mounts and nothing else.
- Fedora/RHEL path differences (`/var/log/secure`) are a config value,
  format differences are a fixtures problem (tracked as known unknown).
- The `Source` interface (`Lines() <-chan RawLine`) keeps demo mode,
  files, and journald symmetric.
