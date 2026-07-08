# ADR-001: v1 scope is sshd + fail2ban translation, not generic monitoring

Date: 2026-07-08
Status: accepted (user-approved direction "A", 2026-07-08)

## Context

The original one-liner ("notify home-server access with a cute UI") is too
close to solved spaces: NetAlertX owns LAN device discovery, ntfy+PAM
tutorials own raw SSH notification delivery, Dozzle/GoAccess own generic
log viewing (see docs/research/2026-07-08-competitive-landscape.md).
The unowned gap is translating security events into language a
non-technical family member can read, with aggregation that avoids
notification fatigue.

## Decision

v1 ingests exactly two sources — OpenSSH sshd logs (auth.log format) and
fail2ban logs — and owns three responsibilities:

1. classify events (login success/failed attempts/ban/unban);
2. aggregate bot noise (per-IP buckets, daily digest);
3. translate to warm household language (ja/en) shown in one timeline UI
   and pushed via ntfy.

Explicit non-goals for v1: device discovery, generic log viewing, nginx or
other web access logs, two-way controls (allow/deny buttons), user
authentication/multi-tenancy, LLM-generated text.

## Consequences

- Small, defensible product surface; parser work is bounded by fixtures.
- The "front door" metaphor stays honest: SSH is the door v1 watches.
- Parser layer must still be a plugin-shaped interface so nginx (v2) does
  not require rework.
- Fedora/RHEL `/var/log/secure` variants are deferred; Debian/Ubuntu-style
  auth.log is the v1 reference target (see DESIGN "known unknowns").
