# doorlog v1 design

Status: draft for review · Date: 2026-07-08 · Owner: Saber5656
Decisions: see `docs/decisions/ADR-001..006`. Research: `docs/research/`.
Issue breakdown: `docs/ISSUE_PLAN.md` + `docs/issues/`.

## 1. Product definition

**doorlog is a friendly doorkeeper for your home server.** It watches
sshd and fail2ban logs and tells your household what happened at the
door — in warm, plain language instead of raw log lines — via one cute
timeline UI and calm push notifications.

| Raw (what admins see today) | doorlog (what the family reads) |
|---|---|
| `Failed password for root from 185.220.101.5 port 22 ssh2` ×47 | ja: 「知らない人が裏口を 47 回ノックしました。全部締め出し済み。対応不要です」 / en: "A stranger knocked on the back door 47 times. All locked out — nothing to do." |
| `Accepted publickey for yasushi from 192.168.1.20` | ja: 「💻 やすしの MacBook から入りました」 / en: "💻 Yasushi's MacBook came in." |

Positioning (ADR-001): a **translation + aggregation layer** on top of
existing plumbing. Delivery → ntfy. Blocking → fail2ban. Device
discovery → NetAlertX's territory (non-goal). doorlog owns meaning.

Success criteria for v1:

1. A reader of the README GIF understands the product in ~5 seconds.
2. `docker compose up` + two `:ro` log mounts = working install.
3. A household member with zero Linux knowledge can read every string
   the UI and notifications produce.
4. At most ~1 push/day under normal bot noise; unknown-device logins
   always push immediately.

## 2. Non-goals (v1)

| Non-goal | Why / where it lives instead |
|---|---|
| Two-way controls (allow/deny, intercom) | Needs auth + privilege model; breaks the size budget |
| LAN device discovery | NetAlertX |
| nginx/web access logs | v2; parser interface is plugin-shaped for it |
| Generic log viewer | Dozzle/GoAccess |
| User accounts / multi-tenancy | LAN-only assumption, documented |
| LLM-generated text | v2 opt-in (ADR-003) |
| Windows/macOS hosts | Linux hosts only in v1 |

## 3. Architecture

```
                     ┌────────────────────────── Go binary ──────────────────────────┐
 /hostlogs/auth.log ─┤ ingest(tail) → parse(sshd) ─┐                                 │
 /hostlogs/fail2ban ─┤ ingest(tail) → parse(f2b) ──┤→ aggregate → enrich → store ────┤→ SQLite (/data)
 testdata/demo/*    ─┤ ingest(demo replay) ────────┘      (buckets)  (device,geo)    │
                     │                                                    │           │
                     │                 ┌── digest scheduler (daily) ──────┤           │
                     │                 ↓                                  ↓           │
                     │              translate(locale yaml) ──→ notify(ntfy POST)      │
                     │                 ↓                                              │
                     │              HTTP API + SSE  ←── embedded Svelte UI (go:embed) │
                     └────────────────────────────────────────────────────────────────┘
```

Pipeline stages are separate packages communicating via channels; every
stage is unit-testable with a fake clock. Store-first: an event is
persisted before any notification attempt (ADR-005).

### 3.1 Repository layout

```
cmd/doorlog/main.go        wiring, flags, graceful shutdown
internal/config/           yaml + env loading, defaults, validation
internal/ingest/           Source interface, file tailer, demo replayer
internal/parse/            Parser interface, sshd, fail2ban, timestamps
internal/event/            Event model, types, severity rules
internal/aggregate/        failed-attempt buckets, dedup windows
internal/enrich/           device matcher, geo (private/mmdb)
internal/translate/        locale loading, template render, tone tests
internal/notify/           ntfy client, delivery rules, retry
internal/digest/           daily stats rollup + scheduler
internal/store/            sqlite (modernc.org/sqlite), migrations
internal/server/           http api, sse, static embed
web/                       Svelte 5 + Vite app (built into internal/server/dist)
locales/ja.yaml, en.yaml   translation templates (embedded, overridable)
testdata/authlog/*.log     parser fixtures
testdata/fail2ban/*.log    parser fixtures
testdata/demo/             demo-mode sample logs
docs/                      this design, ADRs, issues
Dockerfile, docker-compose.yml, .github/workflows/ci.yml
```

## 4. Event model

```go
type Event struct {
    ID          string    // ULID
    TS          time.Time // event time (bucket start for aggregates)
    Type        string    // see table
    Severity    string    // "ok" | "notice" | "alert"
    User        string    // may be attacker-controlled text
    IP          string
    DeviceID    *int64    // matched known device
    Private     bool      // RFC1918/loopback/link-local
    Country     string    // ISO 3166-1 alpha-2, "" if unknown
    Count       int       // >=1; >1 only for ssh.failed_attempts
    WindowStart *time.Time
    WindowEnd   *time.Time
    RawSample   string    // one representative raw line (truncated 512B)
    Source      string    // "authlog" | "fail2ban" | "demo"
}
```

### 4.1 Event types and severity

| Type | Produced by | Severity | Delivery class (ADR-004) |
|---|---|---|---|
| `ssh.login.success` | sshd `Accepted ...` | known device → `ok`; unknown → `alert` | known → timeline-only; unknown → **immediate** |
| `ssh.failed_attempts` | sshd `Failed ...` / `Invalid user ...`, aggregated | `notice` (private IP count=1 → `ok`) | digest |
| `fail2ban.banned` | fail2ban `Ban <ip>` | `notice` | digest (immediate if `notify.immediate_ban: true`) |
| `fail2ban.unbanned` | fail2ban `Unban <ip>` | `ok` | timeline-only |

Rationale: no separate "bruteforce" type — `Count` on
`ssh.failed_attempts` carries intensity, and templates vary wording by
count (ADR-004). `Invalid user` lines fold into the same buckets.

## 5. Parsers

`Parser` interface: `Parse(line RawLine) (RawEvent, bool)`. A parser
returns `false` for lines it does not recognize (they are dropped,
counted in a `parse_ignored_total` metric log).

### 5.1 sshd (auth.log format)

Line = syslog prefix + `sshd[pid]: ` + message. Both timestamp styles
must parse:

- RFC3164 (`Jul  8 03:12:44`, no year): regex
  `^(?P<ts>[A-Z][a-z]{2}\s+\d{1,2} \d{2}:\d{2}:\d{2}) (?P<host>\S+) sshd\[\d+\]: (?P<msg>.*)$`
  Year completion: assume current year; if result > now+24h, subtract
  one year (December→January boundary). Timezone: host TZ (config).
- ISO/RSYSLOG_FileFormat
  (`2026-07-08T03:12:44.123456+09:00 host sshd[123]: ...`): same shape
  with RFC3339 timestamp.

Message patterns (Go regexp, anchored):

| Pattern | → RawEvent |
|---|---|
| `^Accepted (publickey|password|keyboard-interactive/pam) for (\S+) from (\S+) port \d+ ssh2(?:: (\S+) (\S+))?` | login.success {user, ip, method, keytype?, fingerprint?} |
| `^Failed (password|publickey|keyboard-interactive/pam) for (?:invalid user )?(.+?) from (\S+) port \d+ ssh2` | login.failed {user, ip} |
| `^Invalid user (.*?) from (\S+)(?: port \d+)?$` | login.failed {user, ip} (dedup note: the paired `Failed password for invalid user` line for the same attempt is the one counted; `Invalid user` alone counts only if no `Failed` line follows within 5 s — simplification: parser emits both, aggregator dedups by (ip, second) keeping max 1 per second) |
| `^Connection closed by (?:authenticating|invalid) user (\S+) (\S+) port \d+ \[preauth\]$` | login.failed {user, ip} (covers clients that give up before N tries) |

Everything else (`banner exchange`, `Disconnected`, `pam_unix` session
lines, ...) is ignored in v1. Fixtures: `testdata/authlog/` must include
at least 14 lines — the four patterns above in both timestamp styles,
an IPv6 source, a username containing spaces/UTF-8 from `Invalid user`,
a >8 KB line (truncated safely), and unrecognized lines asserting `false`.

### 5.2 fail2ban

```
^(?P<ts>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}),\d+ fail2ban\.actions\s+\[\d+\]: NOTICE\s+\[(?P<jail>[^\]]+)\] (?P<action>Ban|Unban|Restore Ban) (?P<ip>\S+)$
```

`Restore Ban` (re-ban after fail2ban restart) maps to `fail2ban.banned`
with a template variant that avoids double-alarming. Jail name is kept
in RawEvent for v2 but not translated in v1 (only `sshd` jail expected).
Fixtures: Ban/Unban/Restore Ban + non-NOTICE lines ignored.

### 5.3 Robustness rules (both parsers)

- Input lines longer than 8 KB are truncated before regex (ReDoS/memory
  guard, threat T4).
- Parsed usernames are attacker-controlled: stored verbatim, but
  control characters (C0/C1) and newlines are stripped at parse time
  (threat T2), and length-capped at 64 runes.

## 6. Aggregation (`ssh.failed_attempts`)

- Key: source IP. Bucket opens at first failed attempt, `TS` = first
  attempt time, `WindowStart/End` track the span, `Count` increments.
- Bucket closes when: 15 min elapsed since `WindowStart`, or 30 min idle
  since last increment, or the daily digest fires — whichever first.
  While open, the stored event row is updated in place (`Count`,
  `WindowEnd`), and an SSE `event.updated` is emitted so the UI counter
  ticks live.
- Per-second dedup: max 1 increment per (ip, second) to absorb the
  `Invalid user` + `Failed password` pair for one attempt.
- Severity: `notice`; special case count=1 AND private IP → `ok`
  (family typo, worded gently).
- Fake-clock unit tests cover: open→increment→idle-close, 15-min close,
  digest force-close, per-second dedup, two IPs in parallel.

## 7. Enrichment

### 7.1 Known devices

Table `devices` (§10). Match order for `ssh.login.success`:

1. `fingerprint` exact match (from `Accepted publickey ... SHA256:xxx`);
2. else (`user` matches AND `ip` matches — `ip` may be exact or CIDR).

Match → `DeviceID` set, severity `ok`, templates use `{device_name}` and
`{device_emoji}`. No match → unknown-device flow (`alert`, immediate
push, UI card offers "name this device" which POSTs to `/api/devices`
pre-filled with user/ip/fingerprint).

### 7.2 Geo

- `Private` = RFC1918 + loopback + link-local + CGNAT (100.64/10, covers
  Tailscale) — CGNAT renders as {geo.tailnet} wording, distinct from LAN.
- Optional `geo.mmdb_path` (user-supplied GeoLite2-Country.mmdb; license
  requires the user's own MaxMind account — doorlog never bundles it).
  Present → `Country` filled via `oschwald/geoip2-golang`.
- Country display names come from the locale files (`countries:` map,
  ~30 common codes + fallback to the raw code). Absent mmdb → generic
  "somewhere outside" wording.

## 8. Translation layer

Renderer: Go `text/template` over locale YAML, strict mode — a missing
key or unused placeholder fails CI (golden tests in both locales).
Message key = `<event.Type>.<variant>.<context>` where context ∈
`timeline | push | push_title` and variant is resolved by rules
(e.g. `known/unknown` for login, `one/few/many` for counts: 1 / 2–9 / 10+).

`locales/ja.yaml` (authoritative starting content, abbreviated here —
the full file ships in Issue 07 and is the single source of tone):

```yaml
events:
  ssh.login.success:
    known.timeline: "{device_emoji} {device_name}から{user}さんが入りました"
    unknown.timeline: "知らない端末から {user} としてログインがありました（{ip}・{place}）"
    unknown.push_title: "🚪 知らない端末からログイン"
    unknown.push: "{user} として{place}からログインがありました（{ip}）。心当たりがなければ、パスワード変更などの対応を検討してください。"
  ssh.failed_attempts:
    one_private.timeline: "家の中（{ip}）で {user} さんが鍵を間違えました。よくあることです"
    one.timeline: "{place}から入ろうとして失敗した人がいました（{ip}）"
    few.timeline: "{place}から {count} 回ノックがありました（{ip}）"
    many.timeline: "{place}から {count} 回しつこくノックされました（{ip}）"
  fail2ban.banned:
    default.timeline: "しつこい訪問者（{ip}・{place}）を出入り禁止にしました"
    default.push_title: "🛡 出入り禁止にしました"
    default.push: "しつこい訪問者（{ip}・{place}）を自動で出入り禁止にしました。対応は不要です。"
    restore.timeline: "再起動したので、{ip} の出入り禁止を掛け直しました"
  fail2ban.unbanned:
    default.timeline: "{ip} の出入り禁止期間が終わりました"
digest:
  title: "🏠 今日の玄関まとめ"
  none: "今日は誰も玄関に来ませんでした。静かな一日でした 🌿"
  summary: "{ip_count} か所から合計 {attempt_count} 回ノックされました。{banned_count} 件を出入り禁止にし、家は守られています 🛡"
  logins_line: "家族の出入りは {login_count} 回でした"
door_status:
  guarded: "見張り中・異常なし"
  knocked: "今日 {count} 回ノックされました（対応済み）"
  check: "確認してください"
geo:
  private: "家の中"
  tailnet: "自分のネット経由"
  unknown: "どこか外"
  with_country: "{country}のあたり"
countries: { US: アメリカ, CN: 中国, RU: ロシア, JP: 日本, KR: 韓国, DE: ドイツ, ... }
```

`locales/en.yaml` mirrors every key (`"A stranger knocked {count}
times..."` etc.).

Tone guide (enforced in review + golden tests):

1. Warm and concrete; zero jargon (no "SSH", "IP address" alone —
   parenthesized technical detail is allowed after the plain sentence).
2. If the system already handled it, say so and say "no action needed".
3. If action might be needed (unknown login), say what to do in one
   sentence, without panic words.
4. Never blame a family member. Never joke about a real alert.

UI chrome strings (buttons, headings) live in the same locale files
under `ui:` and are served to the frontend via `GET /api/locale`.

## 9. Notifications (ntfy)

- `POST {ntfy.server_url}/{ntfy.topic}`; headers: `Title` (push_title),
  `Priority` (alert→`high`, digest/notice→`default`), `Tags`
  (`door`+severity tag); body = translated push text. Optional
  `Authorization: Bearer $DOORLOG_NTFY_TOKEN` (env only, never in yaml —
  secrets stay out of files the user might commit).
- Delivery rules = ADR-004 table; evaluated after store commit.
- Retry: 3 attempts, backoff 5 s/30 s/2 min; terminal failure recorded
  in `notify_failures` (visible in Settings UI with the error string).
- `POST /api/notify/test` sends a localized test message.
- `notify.include_ip: false` replaces `{ip}` with "(hidden)" wording in
  push payloads only (timeline unaffected) — see ADR-005 privacy note.

## 10. Storage (SQLite)

`modernc.org/sqlite`, WAL mode, busy_timeout 5 s. Migrations: embedded
sequential SQL files applied at boot (`schema_migrations` table).

```sql
CREATE TABLE events (
  id TEXT PRIMARY KEY, ts TEXT NOT NULL, type TEXT NOT NULL,
  severity TEXT NOT NULL CHECK(severity IN ('ok','notice','alert')),
  user TEXT NOT NULL DEFAULT '', ip TEXT NOT NULL DEFAULT '',
  device_id INTEGER REFERENCES devices(id) ON DELETE SET NULL,
  private INTEGER NOT NULL DEFAULT 0, country TEXT NOT NULL DEFAULT '',
  count INTEGER NOT NULL DEFAULT 1, window_start TEXT, window_end TEXT,
  raw_sample TEXT NOT NULL DEFAULT '', source TEXT NOT NULL,
  backfilled INTEGER NOT NULL DEFAULT 0,  -- 1 = ingested via backfill: timeline-only, excluded from digests
  created_at TEXT NOT NULL, updated_at TEXT NOT NULL
);
CREATE INDEX idx_events_ts ON events(ts DESC);
CREATE INDEX idx_events_open_bucket ON events(type, ip, window_end) WHERE type='ssh.failed_attempts';

CREATE TABLE devices (
  id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL,
  emoji TEXT NOT NULL DEFAULT '💻', user TEXT NOT NULL DEFAULT '',
  ip TEXT NOT NULL DEFAULT '',            -- exact or CIDR
  fingerprint TEXT NOT NULL DEFAULT '',   -- SHA256:...
  created_at TEXT NOT NULL
);

CREATE TABLE digests (
  id INTEGER PRIMARY KEY AUTOINCREMENT, date TEXT NOT NULL UNIQUE,
  stats_json TEXT NOT NULL,   -- {ip_count, attempt_count, banned_count, login_count, top_ips:[{ip,count,country}]}
  sent_at TEXT                -- null = not pushed (rendered per-locale on read)
);

CREATE TABLE notify_failures (
  id INTEGER PRIMARY KEY AUTOINCREMENT, ts TEXT NOT NULL,
  event_id TEXT, error TEXT NOT NULL
);

CREATE TABLE settings (key TEXT PRIMARY KEY, value TEXT NOT NULL);
-- includes ingest watermarks: key='watermark:<path>', value='{"inode":N,"offset":N}'
```

Retention: nightly prune of `events` older than `retention_days`
(default 90) and `digests` older than 400 days.

## 11. HTTP API

JSON, no auth (v1, LAN assumption — §14). Base `/api`. Errors:
`{"error": {"code": "...", "message": "..."}}` with 4xx/5xx.

| Endpoint | Behavior |
|---|---|
| `GET /api/events?limit=50&before=<ulid>&severity=a,b` | Timeline page, translated `text` included per current locale, newest first, cursor pagination |
| `GET /api/summary/today` | `{door_status:"guarded|knocked|check", attempts_today, sources_today, banned_today, logins_today, last_event_ts}` — `check` iff an `alert` event in last 24 h is not "resolved" by registering the device |
| `GET /api/digests?limit=14` | Rendered digests for timeline interleave |
| `GET /api/devices` / `POST` / `PATCH /:id` / `DELETE /:id` | Known device CRUD; POST body `{name, emoji, user, ip, fingerprint}` (any subset of the three matchers, at least one required) |
| `GET /api/settings` / `PUT` | Whitelisted keys only: `locale`, `digest_time`, `notify.*`; ntfy token never readable back (write-only, env-preferred) |
| `POST /api/notify/test` | Sends test push, returns delivery result |
| `GET /api/notify/failures?limit=10` | Recent delivery failures (ts, error) for the settings page |
| `GET /api/locale` | All `ui:` + `events:` strings for the active locale (frontend i18n) |
| `GET /api/stream` | SSE: `event.created`, `event.updated`, `summary.changed` |
| `GET /healthz` | 200 + `{version, demo:bool}` |

State-changing endpoints require `Origin` header matching the request
host or absence thereof (curl) — cheap CSRF guard without auth (T3).

## 12. Web UI (Svelte 5)

One SPA, hash routing: `#/` timeline, `#/devices`, `#/settings`.
Phone-first (family checks on phones), responsive to desktop.

### 12.1 Timeline (`#/`)

- Header card "today at your door": big door illustration (inline SVG,
  three states: green/guarded, amber/knocked, red/check — matches
  `door_status`), status line + today's numbers from `/api/summary/today`.
- Feed: date separators; event cards = round icon (severity-tinted
  background, emoji), translated sentence, relative time. `alert` cards
  carry a "この端末を登録 / name this device" button.
- Digest cards interleave at their date position.
- SSE: new event → door SVG plays a small "knock" wiggle (CSS keyframes,
  ~0.6 s, `prefers-reduced-motion` respected); open-bucket count ticks
  on `event.updated`.
- Infinite scroll upward via cursor pagination.

### 12.2 Devices / Settings

- Devices: list (emoji, name, matchers), add/edit/delete modal.
- Settings: locale (ja/en), digest time, ntfy server/topic, immediate-ban
  toggle, include-IP toggle, test-notification button, last delivery
  failures list.

### 12.3 Visual identity ("cute" made concrete)

- Fonts: M PLUS Rounded 1c (ja) + Nunito (latin), self-hosted woff2
  subsets bundled in the binary — **no external font/CDN requests ever**
  (offline LAN must work; subset licenses: both SIL OFL, NOTICE file).
- Palette: bg cream `#FAF7F2` / dark `#1F1B17`; card white `#FFFFFF` /
  dark `#2A241F`; text `#3D3A36` / dark `#EDE7E0`; green `#7BC496`;
  amber `#F2B84B`; rose `#E37B7B`; borders 1px `#E8E1D8`.
  Light/dark via `prefers-color-scheme` + manual toggle.
- Shape: card radius 16 px, icon circles 40 px, generous whitespace;
  no shadows heavier than `0 1px 3px rgba(0,0,0,.06)`.
- Iconography: emoji-first (🚪🛡🔑💻🌏), one bespoke door SVG. No icon
  font dependency.
- All log-derived strings render as text nodes only — `{@html}` is
  banned in the codebase (CI grep) (T1).

## 13. Configuration

Precedence: env (`DOORLOG_*`) > `/data/doorlog.yml` > defaults.
Full default file (also `configuration.md` source of truth):

```yaml
locale: ja                 # ja | en
listen: ":8090"
data_dir: "/data"
timezone: ""               # empty = host TZ
ingest:
  authlog_path: "/hostlogs/auth.log"
  fail2ban_path: "/hostlogs/fail2ban.log"   # "" disables
  backfill: false
  poll_interval: "1s"
aggregate: { bucket: "15m", idle_close: "30m" }
notify:
  ntfy_url: ""             # e.g. https://ntfy.example.com ("" disables push)
  ntfy_topic: ""
  immediate_ban: false
  include_ip: true
digest: { time: "21:00", skip_empty: false }   # empty-day digest is the calm heartbeat; opt out here
geo: { mmdb_path: "" }
retention_days: 90
demo: false                # or DOORLOG_DEMO=1
```

Secrets: only `DOORLOG_NTFY_TOKEN` (env-only by design; setup docs tell
the user to create tokens themselves).

## 14. Security & privacy

Principles: read-only inputs; outbound = ntfy only; no telemetry;
everything else stays on the host.

| # | Threat | Mitigation |
|---|---|---|
| T1 | Attacker-controlled usernames rendered in UI (stored XSS) | Svelte text interpolation only; `{@html}` banned via CI grep; API returns JSON (no HTML composition server-side) |
| T2 | Log-derived strings injected into push payloads (header/control chars) | Strip C0/C1 + newlines at parse; ntfy values header-encoded; 64-rune cap |
| T3 | No-auth UI abused cross-site | LAN-bind guidance; Origin check on state-changing routes; no cookies at all |
| T4 | Hostile/oversized log lines (ReDoS, memory) | 8 KB line truncation; anchored linear regexes; fuzz test on parsers |
| T5 | Notification storms | Aggregation-first model (ADR-004); notifier rate floor: min 30 s between pushes, overflow folds into digest |
| T6 | SQL injection / path traversal | Prepared statements only; config paths cleaned + must be absolute |

Privacy defaults documented in README: prefer self-hosted ntfy; on
ntfy.sh use an unguessable topic; `include_ip: false` for stricter
households. v1 has no auth → README states "bind to LAN / use a reverse
proxy with auth for remote access" prominently.

## 15. Ingest details

- Tailer: 1 s polling; reopen on inode change; reset on size regression;
  watermark `(inode, offset)` persisted per path after each batch (§10)
  so restarts neither re-notify nor double-count digests.
- Demo mode (`demo: true`): replays `testdata/demo/authlog.log` +
  `fail2ban.log` mapped onto "now" (scenario timestamps are relative
  offsets), at ~40× speed, looping a 24 h scenario that includes: two
  known-device logins, one unknown login, three bot bursts from
  distinct IPs, one ban, digest firing. Demo events carry
  `source: demo` and a UI ribbon shows "demo". Used for the README GIF
  and by anyone evaluating without a server.
- journald adapter: same `Source` interface via
  `journalctl -u ssh -f -o json`; optional issue (wave 3), image gains
  `systemd` client libs only in a `-journald` variant if size demands.

## 16. Distribution

- Multi-stage Dockerfile: `node:22-alpine` (web build) →
  `golang:1.24-alpine` (embed + build, CGO off) → `alpine:3.20` runtime
  (non-root UID 65532, `/data` volume). Image target < 40 MB.
- `docker-compose.yml` (README copy-paste):

```yaml
services:
  doorlog:
    image: ghcr.io/saber5656/doorlog:latest
    ports: ["8090:8090"]
    volumes:
      - /var/log/auth.log:/hostlogs/auth.log:ro
      - /var/log/fail2ban.log:/hostlogs/fail2ban.log:ro
      - doorlog-data:/data
    environment:
      DOORLOG_LOCALE: ja
      DOORLOG_NOTIFY_NTFY_URL: https://ntfy.sh
      DOORLOG_NOTIFY_NTFY_TOPIC: my-unguessable-topic-x7k2
    restart: unless-stopped
volumes: { doorlog-data: {} }
```

- GHCR publish on tag; goreleaser binaries are post-v1.

## 17. QA gates

Per-issue Definition of Done includes its Validation command. CI
(GitHub Actions, on PR): 

1. `go vet ./...` + `golangci-lint run` + banned-pattern grep (`{@html}`).
2. `go test ./...` — parser fixtures, aggregator fake-clock suite,
   translator golden files (ja+en, fail on missing key), notifier rules,
   store migrations up-from-empty and up-from-v1.
3. Parser fuzz (`go test -fuzz=FuzzParse -fuzztime=30s` in nightly, seed
   corpus in PR run).
4. `web/`: `npm run check` (svelte-check) + `npm run build`.
5. Docker build; then smoke: run image with `DOORLOG_DEMO=1`, poll
   `/healthz` until 200, assert `GET /api/events` returns >0 events and
   `GET /api/summary/today` parses.

v1 release acceptance (all must hold):

- [ ] compose file above works on a stock Ubuntu 24.04 host with sshd +
      fail2ban (dogfooding host counts)
- [ ] demo mode produces the README GIF scenario end-to-end
- [ ] all six event templates render correctly in ja and en (golden)
- [ ] unknown-device login pushes within 5 s of the log line appearing
- [ ] digest pushes at the configured time with correct counts
- [ ] README: GIF, 5-minute setup, privacy notes, rsyslog note for
      journald-only hosts

## 18. Known unknowns

| # | Unknown | Plan |
|---|---|---|
| U1 | sshd message variants across distros/versions (e.g. `sshd-session` process name in newer OpenSSH) | Fixture-driven; add lines as encountered; `sshd(-session)?` already in the prefix regex for Issue 03 |
| U2 | fail2ban `dateformat` overrides | Document assumption (default format); ignore-with-metric otherwise |
| U3 | GeoLite2 licensing text in README | Verify exact attribution wording during Issue 06 |
| U4 | Font subsetting pipeline (M PLUS Rounded 1c is large) | Issue 11 validates woff2 subset < 300 KB total; fallback = system rounded stack |
| U5 | journald-only hosts UX | ADR-006; optional Issue 14 |

## 19. v2 backlog (recorded, not planned)

nginx/Caddy access logs · Apprise fan-out · LLM digest prose (opt-in) ·
home illustration scene per family member · PWA + web push · Fedora
`/var/log/secure` fixtures · journald first-class · read-only share link.
