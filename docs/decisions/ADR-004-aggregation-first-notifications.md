# ADR-004: aggregation-first notification model

Date: 2026-07-08
Status: accepted

## Context

Public SSH endpoints receive hundreds to thousands of automated failed
logins per day. Forwarding them 1:1 (what ntfy tutorials do) makes the
app unusable for a family — notification fatigue is the #1 product risk.

## Decision

Three delivery classes, decided by event type + device knowledge:

| Class | What | Default |
|---|---|---|
| Immediate | `ssh.login.success` from unknown device/IP; optionally `fail2ban.banned` | push now, high priority |
| Timeline-only | `ssh.login.success` from known device; `fail2ban.unbanned` | visible in UI, no push |
| Digest | `ssh.failed_attempts` (and banned/unbanned rollup) | one daily push at configured time (default 21:00) |

Failed attempts are never stored line-per-line: the aggregator folds them
into one event per (source IP × 15-minute bucket) with a running count.
The daily digest sums buckets into "N sources, M attempts, K banned".

## Consequences

- The family sees at most ~1 push/day in the common case; genuinely
  alarming events (unknown successful login) always break through.
- Storage stays small even under brute-force storms.
- Bucket parameters (15 min window, close after 30 min idle) are config
  constants with defaults, not user-facing settings, to keep v1 simple.
- A single failed attempt from a private/LAN IP is likely a family typo:
  it still becomes a (count=1) bucket event, worded gently, never an alarm.
