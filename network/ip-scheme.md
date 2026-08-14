# IP Scheme — The Galaxy Homelab

All homelab traffic runs on `10.0.0.0/16`. The house network (`senate`) runs on
`192.168.1.0/24`. A static route on `senate` points `10.0.0.0/16` to `tarkin`'s
WAN IP (`192.168.1.100`), enabling house devices to reach homelab services.

---

## VLAN Subnets

| VLAN | Name | Subnet | Gateway | DHCP Pool | Purpose |
|------|------|--------|---------|-----------|---------|
| 10 | Management | 10.0.10.0/24 | 10.0.10.1 | .100–.200 | Switches, APs, Proxmox host |
| 20 | Trusted | 10.0.20.0/24 | 10.0.20.1 | .100–.200 | Personal devices (falcon, scout, comlink) |
| 30 | Servers | 10.0.30.0/24 | 10.0.30.1 | .100–.200 | VMs and NAS |
| 40 | Family | 10.0.40.0/24 | 10.0.40.1 | .100–.200 | Family devices (optional) |
| 50 | IoT/Guest | 10.0.50.0/24 | 10.0.50.1 | .100–.200 | Smart devices, guests |
| 60 | Security Lab | 10.0.60.0/24 | 10.0.60.1 | .100–.200 | Fully isolated attack/defense lab |

---

## Static IP Assignments

Addresses below `.100` in each subnet are reserved for static assignments.
DHCP pools begin at `.100` to avoid conflicts.

### Infrastructure

| Hostname | Role | IP | VLAN | Notes |
|----------|------|----|------|-------|
| tarkin | OPNsense firewall | 10.0.10.1 (gateway), 10.0.20.1, etc. | All | Gateway on each VLAN |
| tarkin | WAN | 192.168.1.100 | — | Static, DHCP reservation on senate |
| executor | Proxmox hypervisor | 10.0.10.10 | 10 | HTTPS :8006 |
| executor | House management | 192.168.1.225 | — | Static on senate network |
| death-star | Managed switch | 10.0.10.2 | 10 | HTTP :80 |
| holonet | WiFi AP | 10.0.20.3 | 20 | HTTP :80 (management via wired only) |

### Servers (VLAN 30)

| Hostname | Role | IP | Services |
|----------|------|----|----------|
| executor | Proxmox host — storage leg | 10.0.30.2 | vmbr1.30 — host↔NAS NFS (ADR 0009); remote mgmt UI :8006 (ADR 0011) |
| phantom | Tailscale gateway | 10.0.30.10 | Subnet router (10.0.30.0/24) + exit node (ADR 0011) |
| archives | TrueNAS SCALE | 10.0.30.20 | NFS, SMB, storage |
| shipyard | Docker host | 10.0.30.25 | Portainer :9443, Crafty :8443, Minecraft :25565 |
| order66 | Pi-hole | 10.0.30.53 | DNS :53, Web UI :80 |
| cantina | Jellyfin | 10.0.30.50 | Media server :8096 (Phase 4 — see note below) |
| vault | Immich | 10.0.30.40 | Immich :2283 (Phase 5 — see note below) |
| inquisitor | Wazuh SIEM | 10.0.30.60 | SIEM dashboard :443 |

shipyard holds its in-OS static .25 **plus** a matching Kea reservation (added 2026-07-14 —
pool-collision guard + Unbound name registration) after the Phase 3 lease audit found the
NIC also holding a dynamic lease (leftover netplan `dhcp4: true`; removed).

cantina is deliberately DHCP-configured in-OS (netplan `dhcp4: true`) with a Kea
reservation at 10.0.30.50 (`bc:24:11:78:00:83`, VLAN 30, Phase 4) — not static-in-OS
like its VLAN 30 neighbors. Name registration consumes the DHCP exchange, so the lease
is what registers `cantina` in Unbound; a static-in-OS host never appears in it
(network/dns-design.md).

vault follows the same deviation (Phase 5): DHCP in-OS plus a Kea reservation at
10.0.30.40, so the lease registers `vault` in Unbound and the FQDN resolves. The
reservation is also what `/etc/hosts` on vault points its own hostname at, instead of the
installer's `127.0.1.1` (phases/phase-5-vault-immich.md).

### Trusted Devices (VLAN 20)

| Hostname | Role | IP | Notes |
|----------|------|----|-------|
| falcon | Main PC | 10.0.20.10 | Kea reservation — created 2026-07-14 (previously documented but never existed; Phase 3 log) |
| scout | Laptop | DHCP | Linux Mint Cinnamon — OS hostname reconciled to scout 2026-07-14 (was "evil"); moves to VLAN 60 in Phase 7 (ADR 0007) |
| comlink | iPhone | DHCP | Randomized MAC — deliberately no reservation |

### IoT / Guest (VLAN 50)

| Hostname | Role | IP | Notes |
|----------|------|----|-------|
| echo | Amazon Echo | 10.0.50.10 | Kea reservation (2026-07-14) |
| led-strip | Magic Home LED strip | 10.0.50.11 | Kea reservation (2026-07-14) |
| levoit-purifier | Levoit air purifier | 10.0.50.12 | Kea reservation (2026-07-14) |
| levoit-humidifier | Levoit humidifier | 10.0.50.13 | Kea reservation (2026-07-14) |
| fire-tv | Fire TV stick | 10.0.50.14 | Kea reservation — moved from TRUSTED 2026-07-14 (trust-zone violation; Phase 3 log) |

Phones and guest devices keep randomized (locally administered) MACs and are deliberately
NOT reserved — a reservation pinned to a rotating MAC breaks silently.

### Security Lab (VLAN 60)

| Hostname | Role | IP | Notes |
|----------|------|----|-------|
| scout | Attack laptop (Linux Mint) | 10.0.60.10 | Phase 7 — moves from VLAN 20 (ADR 0007); DHCP reservation |
| rogue | Vulnerable VM | 10.0.60.50 | DHCP reservation, fully isolated |

---

## House Network (senate)

| Device | IP | Notes |
|--------|----|-------|
| senate | 192.168.1.1 | Asus GT-AX11000, house router/gateway |
| executor | 192.168.1.225 | Proxmox management, DHCP reservation |
| tarkin WAN | 192.168.1.100 | OPNsense WAN, DHCP reservation |

### senate Static Route

```
Destination: 10.0.0.0
Netmask:     255.255.0.0
Gateway:     192.168.1.100 (tarkin WAN)
```

This route enables all house devices on `192.168.1.x` to reach homelab services
at `10.0.x.x` without double-NAT.

---

## Offsite (not on any lab subnet)

| Hostname | Role | Address | Notes |
|----------|------|---------|-------|
| echo-base | Pi 5 offsite backup appliance (ADR 0010) | DHCP on mom's LAN; reached over the tailnet | Raspberry Pi 5, Ubuntu 24.04 arm64, ethernet only. Deliberately **not** static in-OS |

echo-base sits at mom's house in NJ, behind senate's counterpart on that LAN — it is not on
`10.0.0.0/16` and has no lab IP. It is deliberately DHCP-configured rather than static
because the appliance is meant to stay location-portable: its durable identity is its tailnet
address, not any LAN address, so it survives being moved to a different house without
reconfiguration.

It reaches archives at `10.0.30.20` by accepting phantom's advertised `10.0.30.0/24` subnet
route (`tailscale up --accept-routes` — Linux clients ignore advertised routes unless
explicitly opted in, unlike iOS and macOS). Verified with `ping 10.0.30.20` returning
`ttl=63`: one hop, phantom forwarding. **No new firewall rules, no port forwards, and no
static routes were required**, and this same path keeps working after the lab relocates to
NY. Node key expiry is disabled on echo-base — an unattended appliance with a ~180-day key
expiry is a silent outage timer.

Tailnet addresses are deliberately not recorded in this repo.

---

## Minecraft

Server runs on `shipyard` (Docker container via Crafty Controller).

| Service | Address | Port |
|---------|---------|------|
| Minecraft Java | 10.0.30.25 | 25565 |
| Crafty web UI | https://10.0.30.25:8443 | — |

Port forward: OPNsense NAT → senate port forward → tarkin → shipyard.
World files backed up on falcon at `C:\minecraft_backup\`.
