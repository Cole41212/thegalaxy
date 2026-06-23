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
| Proxmox host (executor) | auth, kernel, task, cluster | syslog + Wazuh agent |
| shipyard | host auth + Docker/container logs | Wazuh agent + Docker log driver |
| archives (TrueNAS) | auth, SMART/ZFS events, share access | syslog |
| death-star (switch) | port/link, auth | syslog (if supported) |
| falcon / scout | endpoint security events | Wazuh agent |

## Highest-value signals
- **Firewall block logs** — denied inter-VLAN attempts = early lateral-movement signal.
- **DNS query logs (Pi-hole)** — per-client; catches malware C2 / DNS exfil. This is why
  clients point at Pi-hole directly (see `dns-design.md`).
- **Auth logs** — failed/odd logins across hosts.
- **ZFS/SMART events** — drive health before failure.

## Architecture
    sources → syslog/agents → inquisitor (Wazuh manager + indexer, 10.0.30.100)
- Wazuh data/indices on SSD #2 (`ssd-inquisitor`) so log growth never starves the NVMe.
- Dashboards surfaced on panel-1/panel-2 (Phase 6).

## Phasing note (dependency)
Wazuh is Phase 6, but Phases 3–5 build the sources worth logging. Option: stand up a
lightweight syslog sink (or inquisitor early in syslog-only mode) so firewall/DNS/auth
history accrues before detection rules are written. Decide when Phase 3 lands order66.

## Retention
Tune Wazuh retention to SSD #2 capacity; archive older indices to the `holocron` pool if
longer history is wanted.