# Telemetry & Logging

The SIEM (inquisitor / Wazuh, Phase 6) is only as good as what feeds it. This document
defines the log pipeline now so sources can start shipping early and accumulate history
before the SIEM exists.

## Principle
Centralize logs from every meaningful source into one place. Start with a syslog target;
add Wazuh agents where an OS allows. The earlier sources ship, the more history the SIEM
has to work with on day one.

## Sources (priority order)
| Source | Logs | Method |
|---|---|---|
| tarkin (OPNsense) | firewall pass/block, system, VPN, DHCP | remote syslog |
| order66 (Pi-hole) | per-client DNS queries + blocks | syslog / log export |
| phantom (Tailscale gateway) | auth, sshd, tailscaled state | syslog / Wazuh agent (Phase 6) |
| Proxmox host (executor) | auth, kernel, task, cluster | syslog + Wazuh agent |
| shipyard | host auth + Docker/container logs | Wazuh agent + Docker log driver |
| vault (Immich) | host auth + Docker/container logs, Immich auth + admin actions | Wazuh agent + Docker log driver (Phase 6) |
| cantina (Jellyfin) | host auth, Jellyfin auth + playback | Wazuh agent (Phase 6) |
| archives (TrueNAS) | auth, SMART/ZFS events, share access | syslog |
| death-star (switch) | port/link, auth | syslog (if supported) |
| falcon / scout | endpoint security events | Wazuh agent |

## Highest-value signals
- **Firewall block logs** — denied inter-VLAN attempts = early lateral-movement signal.
- **DNS query logs (Pi-hole)** — per-client; catches malware C2 / DNS exfil. This is why
  clients point at Pi-hole directly (see `dns-design.md`).
- **DNS reply codes (Pi-hole)** — a SERVFAIL-ratio spike means the resolver path is broken
  even when users notice nothing: the Phase 3 zero-upstream incident ran invisibly because
  clients lived on the DHCP fallback. Phase 6 candidate rule from that incident.
- **Auth logs** — failed/odd logins across hosts.
- **ZFS/SMART events** — drive health before failure. Demonstrated on 2026-08-02, when the
  weekly scrub faulted a drive on a latent unreadable sector (Phase 5). The same incident
  exposed the gap that matters more than the signal: Proxmox alert mail was not being
  delivered at all, so the pool-degraded notification was never received and the fault was
  found by hand days later. **An alert channel nobody has tested is not a control** —
  verifying delivery end to end is a Phase 6 prerequisite, not a nice-to-have.

## Architecture
    sources → syslog/agents → inquisitor (Wazuh manager + indexer, 10.0.30.60)
- Wazuh data/indices on SSD #2 (`ssd-inquisitor`) so log growth never starves the NVMe.
- Dashboards surfaced on panel-1/panel-2 (Phase 6).

## Phasing note (dependency)
Wazuh is Phase 6, but Phases 3–5 built the sources worth logging. The plan was to decide on
a lightweight interim syslog sink (or inquisitor early in syslog-only mode) when Phase 3
landed order66, so firewall/DNS/auth history would accrue before detection rules were
written. That decision was never taken, and Phases 3, 4, and 5 have all shipped without a
central sink — so none of that history exists. Either stand the sink up now, or accept
explicitly that Phase 6 begins with no back-history to tune rules against.

## Retention
Tune Wazuh retention to SSD #2 capacity; archive older indices to the `holocron` pool if
longer history is wanted.