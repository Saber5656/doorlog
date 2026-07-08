# ADR-005: delegate push delivery to ntfy; doorlog only composes messages

Date: 2026-07-08
Status: accepted

## Context

Notification delivery (apps, push infra, per-service adapters) is a
commodity already solved by ntfy (self-hostable, simple HTTP) and by
Apprise (80+ services). Building or bundling delivery would grow scope
without differentiating.

## Decision

- v1 notifier speaks plain ntfy HTTP: `POST {server_url}/{topic}` with
  `Title`, `Priority`, `Tags` headers; optional bearer token from env.
- doorlog's job ends at composing the (translated) title/body/priority.
- Apprise or other channels: v2 candidates behind the same Notifier
  interface. No SMTP, no LINE, no Slack in v1.

## Consequences

- Zero push infrastructure to maintain; works with ntfy.sh or self-hosted.
- Privacy caveat documented in README and settings: on public ntfy.sh,
  topic names are effectively capability URLs and payloads transit a
  third-party server → recommend self-hosted ntfy or unguessable topic;
  `notify.include_ip` setting (default true) lets users strip IPs from
  push payloads.
- If ntfy is unreachable, notifier retries with backoff and surfaces the
  failure in the UI settings page; events are never lost (store-first).
- The delivery endpoint (`ntfy_url`/`ntfy_topic`) and token are
  config-file/env only — deliberately NOT settable through the no-auth
  API/UI, so the settings surface cannot be turned into an egress/SSRF
  primitive (DESIGN threat T7).
