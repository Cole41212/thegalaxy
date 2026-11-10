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
- **Offsite copy is outside the primary site's metropolitan area — improved by the 2026-08-26
  relocation, not eliminated as a correlated risk.** echo-base sits at a separate physical
  site: different building, different household, different ISP, different power utility. The
  relocation moved the primary site out of the offsite appliance's metro area, which closes the
  specific gap the earlier version of this entry flagged. The offsite tier now survives
  localized events — structure fire, burst pipe, theft, a substation or utility-level outage —
  that were previously capable of destroying both copies in a single incident.

  It does **not** deliver full regional independence. The two sites are still close enough
  that one large weather system, or one multi-day regional grid event, could plausibly affect
  both. Correlated regional risk is therefore **reduced, not eliminated**, and is accepted at
  that level. Genuine regional separation would need an offsite tier in a different region
  entirely — a different appliance location, or a cloud tier — and is not planned.

  The design prediction held as far as it has been tested: this entry previously stated that a
  relocation would increase separation "with no reconfiguration," and echo-base needed no
  changes, because replication rides the tailnet and the tailnet is location-independent.
  **Not yet verified:** that the first post-move replication pull landed, and how its
  throughput compares with the 2026-08-14 seed (~76 MiB/s) now that the path crosses a
  different ISP at the primary end — a slower pull lengthens the effective RPO. Tracked in
  [runbooks/site-relocation.md](../runbooks/site-relocation.md).
- **Physical access to echo-base is now a remote-hands dependency.** The relocation traded
  proximity for separation, and that trade has a cost worth naming. The offsite appliance was
  previously close enough to attend in minutes; a visit now has to be planned. A failed drive,
  a wedged USB enclosure — the ADR 0010 falsifier — or a power event the Pi does not recover
  from now needs either a trip or a household member acting on instructions. This compounds
  the existing `pibackup` exposure rather than sitting beside it: the delegated credential was
  already on hardware outside physical control, and now the recovery path is too. Accepted,
  because the separation it buys is worth more than the proximity it costs. Candidate
  mitigations if it ever bites: a smart plug or networked PDU on the Pi for remote
  power-cycling, and a one-page printed recovery card left at the site.
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
- **Wazuh agents run as root on every monitored host, and the manager can configure them.**
  New exposure, accepted at Phase 6 (ADR 0020). The agent must run as root to do its job — FIM
  hashes files no unprivileged user may read, syscollector enumerates every process and
  listening socket, and log collection reads `auth.log` and its equivalents. Because the
  manager can push configuration to agents, **compromise of inquisitor yields root across the
  lab, plus the ability to delete the evidence of how it happened.** That places inquisitor
  alongside executor and tarkin in the assets list above, and it is a category the lab did not
  previously have: no single component held lab-wide root reach before this. Mitigations, all
  structural rather than procedural: `remote_commands` stays **disabled** (the default), which
  is what stops a manager-pushed `agent.conf` from executing arbitrary commands — and is why
  the Immich dump-freshness check is configured locally on vault instead; **active response is
  disabled globally**, because auto-blocking in a lab whose router is a VM on the same host as
  the SIEM is a self-inflicted lockout (revisit in Phase 7, on the attack range); **authd uses
  an enrolment password**, not open registration; `client.keys` is treated as a secret and
  backed up only to the encrypted offsite pool, never to this repo (ADR 0019); and the manager
  is reachable from **VLAN 30 and the tailnet only**. Accepted because the alternative — no
  agents — gives up FIM, inventory, and configuration assessment on every host, which is most
  of the visibility design goal 3 exists to deliver. Agents dial out, so no monitored host
  needs an inbound port for this.
- **Device configs have no offsite copy.** `holocron/configs` was never created, so the
  OPNsense XML export and switch config exist only in the local, gitignored
  `config-backups/`. Design goal 4 (recoverability) is therefore met for data but not yet
  for configs — a total site loss would mean rebuilding tarkin and death-star from the
  runbooks rather than restoring them. Tracked in ADR 0010; **scheduled for closure in Phase 6
  under ADR 0019**, which creates the dataset for the Wazuh configuration export and adds it to
  the syncoid replication set, carrying the OPNsense and switch configs along with it.
- **Repository is public.** Software versions, network topology, VLAN design, and firewall
  rules are published deliberately as portfolio content. This is only an aid to an attacker
  in combination with the site's public IP, which does not appear in the repo and cannot be
  derived from it. Personal location detail was removed in the 2026-08-16 sanitization pass
  (phases/phase-5-vault-immich.md).
- Physical security, ISP-level threats, and supply chain are out of scope.