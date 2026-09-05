# Research: competitive landscape for doorlog (2026-07-08)

Status: informing v1 scope. Verified via web search on 2026-07-08.

## Question

Is "notify home-server access events with a friendly UI" viable as an OSS
product, or is the space already solved?

## Findings

### Adjacent tools and what they already solve

| Tool | Space | What it does | Overlap with doorlog |
|---|---|---|---|
| [NetAlertX](https://github.com/netalertx/NetAlertX) (ex PiAlert) | Network device discovery | Continuous ARP/DHCP/Nmap scans, live device inventory, alerts on new/changed devices, 80+ notification services via Apprise | Owns "new device on my LAN" alerting. doorlog must NOT compete here |
| WatchYourLAN | Network device discovery | Lightweight device detection for small networks | Owns the "simple/lightweight" position in the same space |
| ntfy + PAM hook | SSH login notification | One curl call from a PAM script notifies on every SSH login ([Hetzner tutorial](https://community.hetzner.com/tutorials/ssh-notification-with-ntfy/), many blog posts) | Solves raw *delivery*. A tutorial, not a product — no aggregation, no history, admin-oriented wording |
| fail2ban + ntfy action | Intrusion block notification | Ban/unban push notifications ([common homelab setup](https://blog.alexsguardian.net/posts/2023/09/12/selfhosting-ntfy)) | Same: delivery solved, meaning-making absent |
| Dozzle, GoAccess, Grafana+Loki | Log viewing/analytics | Generic log viewers and analytics for admins | Own "log viewer for admins". doorlog must not become a generic log viewer |

### The gap

Every tool above targets the **administrator** and outputs **raw or
technical text** (`Failed password for root from 185.220.101.5 port 22
ssh2`). None of them:

1. translate security events into plain household language a non-technical
   family member can read;
2. aggregate bot noise into calm daily digests (thousands of failed
   attempts per day would be notification hell if forwarded 1:1);
3. present the home server's "front door" as a friendly, glanceable
   timeline.

Precedent that "friendly + simple" wins in an admin-tool space: Uptime Kuma
took the monitoring space from Nagios-era tools largely on UI approachability.

### Positioning decision

doorlog = **translation & aggregation layer on top of existing plumbing**,
not another collector/notifier:

- delivery is delegated to ntfy (self-hostable);
- blocking is delegated to fail2ban;
- device discovery is left to NetAlertX (non-goal);
- doorlog owns: parsing sshd/fail2ban events, classifying, aggregating,
  translating to warm human language (ja/en), and one cute timeline UI.

## Sources

- https://github.com/netalertx/NetAlertX
- https://netalertx.com/
- https://www.virtualizationhowto.com/2025/06/netalertx-self-hosted-network-monitoring-for-home-labs/
- https://ambientnode.uk/local-network-monitoring-with-netalertx-watchyourlan-and-nmap-2
- https://community.hetzner.com/tutorials/ssh-notification-with-ntfy/
- https://ollioddi.dev/blog/homelab-ssh-notifications
- https://blog.alexsguardian.net/posts/2023/09/12/selfhosting-ntfy
- https://bacardi55.io/2024/09/09/receive-a-notification-on-ssh-connection-via-ntfy/
