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
- The personal photo/video library (vault — Immich, ADR 0014) and media (archives).
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
- **Ransomware, or any lab-side compromise, reaching the backups** → the offsite copy is
  *pulled*, never pushed. No lab-side credential can write to or delete from `carbonite`,
  and the credential pointing the other way is read-only (ADR 0010). Local backups and ZFS
  snapshots share the lab's fate by design; the offsite tier is the one that does not.

## Key controls
802.1Q segmentation · default-deny inter-VLAN firewall · per-client DNS filtering (Pi-hole)
+ recursive Unbound · centralized logging → Wazuh (Phase 6) · ZFS RAIDZ2 + local backups +
pull-based encrypted offsite replication (ADR 0010, operational 2026-08-14) · Cloudflare
Tunnel for any public service.

## Accepted risks / out of scope
- **Single-host SPOF:** tarkin, archives, and all services run on one Proxmox box; host loss
  = network loss. Accepted for a lab; mitigated by backups + fast-rebuild runbooks.
- **OEM PSU + Molex splitters** powering 14 drives — monitored; hardening planned.
- **pveproxy listens on all host interfaces:** the Proxmox management UI is reachable on
  the VLAN 30 storage leg (https://10.0.30.2:8006) — accepted as the gateway-pattern
  remote-management path (ADR 0011). Candidate mitigations (pve-firewall or a LISTEN_IP
  bind) are from-home tasks, deferred.
- **Tailnet ACLs stay default allow-all** while the tailnet is single-user (every device
  is mine); the controls are device authorization, key management, and MGMT staying
  structurally unadvertised. Revisit when any non-owner device joins, or at Phase 6
  hardening (ADR 0011).
- **FAMILY/IOTGUEST → cantina :8096 (Jellyfin):** the low-trust VLANs reach exactly one
  TCP port on VM 106 — one-host-one-port pinholes in the DNS-exception pattern
  (Phase 4). A Jellyfin compromise reached this way still lands on a VM that mounts the
  library read-only (ADR 0013).
- **Cleartext HTTP to vault :2283 (Immich):** no domain exists until Phase 8, and
  self-signed certs break Immich's mobile client (upload/playback), so vault serves plain
  HTTP on one URL for both LAN and tailnet (ADR 0017). Exposure is VLAN 30 and TRUSTED
  WiFi; the positioned adversary above (compromised IoT/guest) is contained to VLAN 50 with
  no lab reach, and remote access rides phantom's existing subnet route rather than a new
  tailnet node. Time-boxed: Phase 8 remediates with Caddy + ACME DNS-01 on the portfolio
  domain (zero inbound ports).
- **TRUSTED → archives :445 (SMB media ingest):** ingest from falcon runs as `coalish`,
  a dedicated non-admin account scoped to the media dataset (ADR 0013). No new firewall
  rule was involved — TRUSTED's full access to SERVERS is the documented design (trust
  zones above), so the control here is credential scope, not the network path.
- **Offsite copy is within the same metropolitan area as the primary site.** echo-base sits
  at a separate physical site, so today the offsite tier delivers ransomware isolation,
  hardware-failure independence, and household separation — different building, different
  power, different ISP — but not regional separation. Correlated regional risk — severe
  weather, an extended grid outage, any single area-wide event — is not mitigated by this
  arrangement. A planned relocation will increase geographic separation, at which point the
  same appliance over the same tailnet path delivers it with no reconfiguration.
  TODO(cole): characterize this — accepted risk until relocation, or a tracked gap with a
  target date?
- **`pibackup` is a delegated credential resident on off-site hardware.** The account lives
  on archives but its private key sits on a Pi at another site, outside physical control.
  Blast radius if echo-base is stolen or compromised: **read** access to `holocron/photos`
  and `holocron/media`, and nothing else. The delegation is
  `send,snapshot,hold,bookmark,mount` — `receive`, `create`, `destroy`, and `rollback` are
  withheld, so the credential cannot encrypt, delete, or alter the source. carbonite is
  ZFS-encrypted with a key that never leaves the Pi, so the stolen-hardware case yields
  ciphertext. Accepted deliberately: this is the cost of pull-based replication, and it is
  strictly better than the push alternative, where a *lab-side* credential could destroy
  offsite history (ADR 0010).
- **Device configs have no offsite copy.** `holocron/configs` was never created, so the
  OPNsense XML export and switch config exist only in the local, gitignored
  `config-backups/`. Design goal 4 (recoverability) is therefore met for data but not yet
  for configs — a total site loss would mean rebuilding tarkin and death-star from the
  runbooks rather than restoring them. Tracked in ADR 0010.
- Physical security, ISP-level threats, and supply chain are out of scope.