# ADR-003: rule-based translation with locale files; no LLM in v1

Date: 2026-07-08
Status: accepted (user-approved, 2026-07-08; ja + en both ship in v1)

## Context

The product's core value is turning raw log lines into household
language. An LLM could phrase events flexibly, but home users expect the
tool to be offline, deterministic, free to run, and trustworthy about
security wording. Event variety in v1 is small (a handful of types).

## Decision

- Translation is template-based: locale YAML files (`locales/ja.yaml`,
  `locales/en.yaml`) keyed by event type and context (timeline /
  immediate notification / digest), with named placeholders
  (`{user}`, `{device_name}`, `{ip}`, `{count}`, ...).
- Both `ja` and `en` ship in v1; locale files are the i18n structure from
  day one (retrofitting i18n later was judged more expensive).
- Tone guide lives in DESIGN and is enforced by golden tests: warm,
  no jargon, no blame, explicit "no action needed" when the system
  already handled it, honest uncertainty when it did not.
- LLM-generated digests/summaries are a v2 opt-in, never a v1 dependency.

## Consequences

- Output is deterministic and golden-testable in both locales.
- Adding a language = adding one YAML file (community-friendly).
- Phrasing flexibility is bounded by templates; count-dependent variants
  (1 attempt vs 150 attempts) must be explicit template keys.
