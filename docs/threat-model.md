# Threat Model & Design Goals

## What this lab is
A segmented home network and security lab built to defend a small set of personal/family
assets and to practice blue-team skills. Not production — some availability risks are
knowingly accepted (below).

## Design goals
1. **Contain blast radius** — a compromise in one zone shouldn't reach the others.
2. **Least privilege between zones** — default-deny inter-VLAN, explicit allows only.
3. **Visibility** — per-client DNS and centralized logging so activity is observable.
4. **Recoverability** — versioned configs + local and offsite backups (3-2-1).
5. **No inbound exposure** — nothing internet-reachable except via an outbound-only tunnel.

## Assets
- Personal/family files (vault) and media (archives).
- The hypervisor (executor) and firewall (tarkin) — compromise = whole-network control.
- Credentials, configs, and the integrity of the monitoring pipeline.

## Trust zones
| Zone | VLAN | Trust | Rationale |
|---|---|---|---|
| Management | 10 | highest | switch/AP/hypervisor admin planes |
| Trusted | 20 | high | my own devices; full access |
| Servers | 30 | medium | services; no path back to Trusted |
| Family | 40 | low | internet only, no lab access |
| IoT/Guest | 50 | low | internet only, no lab access |
| Security Lab | 60 | untrusted | isolated; rogue has no internet |
| House (senate) | — | external | upstream/untrusted; reached via static route only |

## Adversaries considered
- **Compromised IoT/guest device** → contained to VLAN 50, internet-only, no lab reach.
- **Compromised internet-facing service** (Phase 8 site) → outbound-only Cloudflare Tunnel,
  no exposed ports/home IP; site host slated for an isolated DMZ VLAN.
- **Lab malware escaping the range** (Phase 7) → SECLAB blocked from 10.0.0.0/16; rogue has
  no internet; only scout (attacker) reaches it.
- **Lateral movement Servers → Trusted** → explicitly blocked at tarkin.

## Key controls
802.1Q segmentation · default-deny inter-VLAN firewall · per-client DNS filtering (Pi-hole)
+ recursive Unbound · centralized logging → Wazuh (Phase 6) · ZFS RAIDZ2 + local & offsite
backups · Cloudflare Tunnel for any public service.

## Accepted risks / out of scope
- **Single-host SPOF:** tarkin, archives, and all services run on one Proxmox box; host loss
  = network loss. Accepted for a lab; mitigated by backups + fast-rebuild runbooks.
- **OEM PSU + Molex splitters** powering 14 drives — monitored; hardening planned.
- Physical security, ISP-level threats, and supply chain are out of scope.