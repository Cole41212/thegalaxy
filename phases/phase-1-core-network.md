# Phase 1 — Core Network

**Goal:** Replace a flat consumer home network with a fully segmented, enterprise-style
VLAN architecture enforced by an OPNsense firewall running as a VM on Proxmox. Establish
isolated network zones for trusted devices, servers, family, IoT, and a dedicated
security lab that cannot reach the real network under any circumstances.

**Status:** ✅ Complete

**Duration:** May–June 2026

---

## Decisions & Rationale

### Why OPNsense as a VM rather than dedicated hardware?

Running OPNsense inside Proxmox on `executor` eliminates the need for a separate physical
firewall appliance, saving hardware cost and desk space. More importantly, it enables
snapshot-based configuration testing — before applying risky changes, a Proxmox snapshot
can be taken and instantly restored if something breaks. The performance overhead of
virtualization is negligible for a homelab routing traffic through a consumer internet
connection. This also establishes a single-server architecture that is highly portable:
moving the lab requires moving one machine.

### Why six VLANs?

The VLAN design reflects real-world network segmentation principles used in enterprise
environments. Each zone has a defined trust level and an enforced inter-zone traffic policy:

| VLAN | Name | Trust Level | Policy |
|------|------|-------------|--------|
| 10 | Management | Infrastructure only | Reachable from Trusted; can reach internet |
| 20 | Trusted | Full access | Unrestricted — personal devices |
| 30 | Servers | Services | Can reach internet; blocked from Trusted |
| 40 | Family | Internet only | Cannot reach any homelab subnet |
| 50 | IoT/Guest | Internet only | Cannot reach any homelab subnet |
| 60 | Security Lab | Fully isolated | Cannot reach any real network |

Segmenting the security lab (VLAN 60) is non-negotiable: Kali Linux attacking vulnerable
VMs must never be able to reach real network devices. The firewall enforces this in
software; the managed switch enforces it in hardware via VLAN membership.

### Why a managed switch with 802.1Q trunking?

Consumer routers typically cannot carry multiple VLANs on a single ethernet port.
A managed switch with 802.1Q support allows one physical cable to carry all six VLANs
simultaneously, each identified by its tag. Without a managed switch, physical network
separation would require multiple NICs and cables for every zone.

### Why a static route on senate rather than double-NAT?

`tarkin` sits behind `senate` (the house router) on a private `192.168.1.x` subnet.
A single static route on `senate` pointing `10.0.0.0/16` to tarkin's WAN IP
(`192.168.1.100`) enables house devices to reach homelab services without hairpinning
traffic through two NAT layers. Double-NAT would break port forwarding, complicate DNS
resolution, and create asymmetric routing that is difficult to debug.

### Why Kea DHCPv4 over ISC DHCP?

OPNsense 26.1 ships with Kea as the default DHCPv4 implementation, replacing the legacy
ISC DHCP daemon. Kea is the modern, actively maintained successor with better multi-subnet
support. An important discovery during Phase 1: `dnsmasq` (OPNsense's combined DNS/DHCP
tool) was running simultaneously and had claimed port 67 before Kea could bind to it.
Disabling dnsmasq immediately resolved DHCP service failures across all six subnets.

### Why dedicated NICs for the final architecture?

The initial build used router-on-a-stick — a single physical NIC carrying all traffic
(WAN and LAN) through a VLAN trunk, with VLAN 99 as a WAN transit path through the
switch. The final dedicated-NIC architecture uses two physical NICs: one for the WAN
connection directly to `senate`, and one for the LAN trunk to `death-star`. This
physically separates WAN and LAN, eliminates VLAN 99, places the switch entirely behind
the firewall on the LAN side where it belongs, and matches how enterprise edge networks
are actually built.

### AI-Assisted Development

This project was built with Claude (Anthropic) as an AI pair programmer and network
engineering advisor. This is an intentional workflow decision reflecting how modern
security and infrastructure work actually gets done — the ability to leverage AI tools
to accelerate learning, troubleshoot complex issues, and produce professional documentation
is a core competency for engineers entering the field today. Claude was used for
architecture decisions, configuration generation, troubleshooting diagnosis, and
documentation. All configurations were applied, tested, and verified by the project
owner on real hardware.

---

## Architecture

### Physical Topology (Final State)

```
                        Internet
                            │
                    ┌───────▼────────┐
                    │     senate      │
                    │ Asus GT-AX11000 │
                    │  192.168.1.1   │
                    │ Static route:  │
                    │ 10.0.0.0/16 →  │
                    │  192.168.1.100 │
                    └───────┬────────┘
                            │ enp0s31f6 (WAN NIC, vmbr0)
                ┌───────────▼──────────────────────┐
                │         executor                  │
                │  Proxmox VE 9.1.1               │
                │  i7, 64GB DDR4, 1TB NVMe         │
                │  192.168.1.225 / 10.0.10.10     │
                │  ┌──────────────────────────┐   │
                │  │  tarkin (VM 100)          │   │
                │  │  OPNsense 26.1.6         │   │
                │  │  WAN: 192.168.1.100      │   │
                │  │  VLAN 10:  10.0.10.1/24  │   │
                │  │  VLAN 20:  10.0.20.1/24  │   │
                │  │  VLAN 30:  10.0.30.1/24  │   │
                │  │  VLAN 40:  10.0.40.1/24  │   │
                │  │  VLAN 50:  10.0.50.1/24  │   │
                │  │  VLAN 60:  10.0.60.1/24  │   │
                │  └──────────────────────────┘   │
                │  enp1s0 (LAN NIC, vmbr1)        │
                └───────────┬──────────────────────┘
                            │ VLAN trunk (802.1Q)
                    ┌───────▼────────┐
                    │   death-star    │
                    │ ZX-SWTGW215AS  │
                    │  10.0.10.2     │
                    └──┬──┬──┬───────┘
                       │  │  │
               Port 1  │  │  │ Port 2    Port 3
               trunk   │  │  │ falcon    holonet
               all     │  │  │ VLAN 20   VLANs
               VLANs   │  │  │           20/40/50
```

### VLAN Sub-interfaces on tarkin

| Interface | Device | VLAN | IP | DHCP Pool |
|-----------|--------|------|----|-----------|
| vlan01 | em0.10 | 10 | 10.0.10.1/24 | 10.0.10.100–200 |
| vlan02 | em0.20 | 20 | 10.0.20.1/24 | 10.0.20.100–200 |
| vlan03 | em0.30 | 30 | 10.0.30.1/24 | 10.0.30.100–200 |
| vlan04 | em0.40 | 40 | 10.0.40.1/24 | 10.0.40.100–200 |
| vlan05 | em0.50 | 50 | 10.0.50.1/24 | 10.0.50.100–200 |
| vlan06 | em0.60 | 60 | 10.0.60.1/24 | 10.0.60.100–200 |

### Firewall Policy Matrix

| Source → Destination | TRUSTED | SERVERS | FAMILY | IOTGUEST | SECLAB | MGMT | Internet |
|---|---|---|---|---|---|---|---|
| **TRUSTED** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **SERVERS** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **FAMILY** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ |
| **IOTGUEST** | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| **SECLAB** | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ |
| **MGMT** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### death-star Switch Port Configuration (Final)

| Port | Device | VLAN Membership | PVID | Notes |
|------|--------|-----------------|------|-------|
| 1 | executor LAN (enp1s0) | Tagged: 10,20,30,40,50,60 | 1 | Trunk |
| 2 | falcon | Untagged: 20 | 20 | Access port |
| 3 | holonet | Tagged: 20,40,50 | 20 | AP trunk |
| 4–8 | (unused) | — | 1 | Available |

### Proxmox Network Configuration

```
# /etc/network/interfaces

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.225/24
    gateway 192.168.1.1
    bridge-ports enp0s31f6
    bridge-stp off
    bridge-fd 0

auto vmbr1
iface vmbr1 inet manual
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

auto vmbr1.10
iface vmbr1.10 inet static
    address 10.0.10.10/24
    post-up ip route add 10.0.0.0/16 via 10.0.10.1
    pre-down ip route del 10.0.0.0/16 via 10.0.10.1
```

---

## Checklist

| Step | Status | Notes |
|------|--------|-------|
| Architecture finalized | ✅ | Dedicated NIC final state |
| Proxmox bridges configured (vmbr0 WAN, vmbr1 LAN) | ✅ | |
| tarkin VM created (ID 100), OPNsense 26.1.6 | ✅ | |
| OPNsense VLAN interfaces (vlan01–06 on em0) | ✅ | |
| All 6 VLAN interfaces /24 with correct IPs | ✅ | |
| Kea DHCPv4 — all 6 subnets, correct DNS per VLAN | ✅ | dnsmasq disabled, auto-collect off |
| Firewall rules — all 6 interfaces | ✅ | Legacy rules interface |
| tarkin WAN static 192.168.1.100 | ✅ | DHCP reservation on senate |
| death-star switch VLANs configured | ✅ | VLAN 99 removed |
| senate static route (10.0.0.0/16 → 192.168.1.100) | ✅ | |
| holonet TL-WA1201 Multi-SSID (VLANs 20/40/50) | ✅ | HoloNet/OuterRim/MosEisley |
| executor Proxmox management on 10.0.10.10 | ✅ | via vmbr1.10 |
| Dedicated NIC architecture (enp0s31f6 WAN, enp1s0 LAN) | ✅ | VLAN 99 eliminated |
| All VLANs verified — correct DHCP, DNS, internet | ✅ | |
| WAN block private networks disabled | ✅ | tarkin faces senate LAN not internet |
| Hardware migration to new all-in-one PC | ✅ | NVMe clone, SSD transplant |
| Proxmox NVMe migration (SanDisk → Samsung 970 EVO) | ✅ | LVM expanded to 465GB |
| Firewall rules migration to Rules [new] | ✅ | Pushed to Phase 2 |

---

## Log

### 2026-05-24 — Project start, architecture decisions
Established overall lab architecture. Key decisions: OPNsense as Proxmox VM, six-VLAN
segmentation, managed switch for 802.1Q enforcement, static route on house router instead
of double-NAT. Created tarkin VM (ID 100), installed OPNsense 26.1.6.

### 2026-05-25 — OPNsense NIC assignment fix
**Issue:** tarkin's VLAN interfaces showed incorrect behavior after initial config.
**Root cause:** vtnet0 and em0 NIC assignments were swapped in Proxmox — em0 had tag=99
(WAN position) and vtnet0 had no tag (LAN position), reverse of what OPNsense expected.
**Fix:** Cleared tag from net0 (em0, LAN), set tag=99 on net1 (vtnet0, WAN) in Proxmox
hardware tab.
**Learning:** VM NIC assignments must match what the guest OS expects; order of NIC
creation in Proxmox matters.

### 2026-05-25 — Kea DHCPv4 silent failure (dnsmasq port conflict)
**Issue:** Kea configured correctly but devices got no DHCP leases. No visible errors.
**Diagnosis:** `sockstat -4 -l | grep :67` showed dnsmasq (PID 20861) holding `*:67`.
Kea log showed repeated "Address already in use" for every VLAN interface.
**Fix:** Disabled dnsmasq in Services → Dnsmasq DNS & DHCP. Kea leases began immediately.
**Learning:** On fresh OPNsense installs dnsmasq starts automatically. Always disable it
before configuring Kea to prevent port 67 conflicts. dnsmasq must stay disabled.

### 2026-05-25 — Switch PVID lockout
**Issue:** After setting death-star port 4 (executor trunk) PVID to 10, Proxmox
became completely unreachable at 192.168.1.225.
**Root cause:** Proxmox management sends untagged frames. PVID 10 classified them into
VLAN 10 which had no path to the house network. Proxmox disappeared from 192.168.1.x.
**Fix:** Power cycled switch (config not flash-saved, reverted). Kept port 4 PVID=1.
**Learning:** Switch trunk ports serving Proxmox should keep PVID=1. VM traffic uses
explicit Proxmox-assigned VLAN tags, making PVID irrelevant for VM frames.

### 2026-05-25 — Asymmetric routing for Proxmox management
**Issue:** Connections to 10.0.10.10:8006 from falcon timed out despite firewall
logs showing traffic passing.
**Root cause:** Proxmox had no route for 10.0.20.x. Responses defaulted to gateway
192.168.1.1 (senate) which dropped them. Classic asymmetric routing.
**Fix:** Added `post-up ip route add 10.0.0.0/16 via 10.0.10.1` to vmbr1.10 interface.
**Learning:** In split-managed environments, explicit static routes prevent host OS from
routing responses through a gateway that doesn't know the destination subnets.

### 2026-05-26 — Access point hardware iterations
Three APs tested before finding a working solution:
1. **Asus RT-AC68U** — BCM4708 chip; per-SSID VLAN in AP mode unsupported.
2. **Asus RT-AX1800S** — MediaTek chip; same limitation, no Merlin support.
3. **TP-Link TL-WA1201** — Multi-SSID mode with per-SSID VLAN; works correctly.
**Learning:** Verify AP per-SSID VLAN capability before purchase. Consumer routers
repurposed as APs often cannot do this correctly.

### 2026-05-26 — Hardware migration to new all-in-one PC
**Context:** Consolidated onto single machine (i7, 64GB DDR4, 1TB NVMe 970 EVO,
GPU, 6×4TB HDDs, 3× NICs, two 512GB SSDs).
**Issue:** After SSD transplant, vmbr0 couldn't come up — interfaces file referenced
`eno1` (old NIC name). New machine NICs are `enp0s31f6` and `enp1s0`.
**Fix:** Updated bridge-ports reference in /etc/network/interfaces.
**Learning:** NIC names are assigned by udev based on PCI bus location. Hardware moves
almost always require updating interface names.

### 2026-05-26 — Proxmox NVMe migration
Cloned Proxmox from SanDisk SATA SSD (238GB) to Samsung 970 EVO NVMe (500GB) using dd.
Had to remove old drive before pvresize/lvextend due to duplicate LVM UUID conflict —
with only one drive present, LVM had no ambiguity. LVM data volume expanded to use full
465GB. Boot order updated to NVMe in BIOS. Old SSD removed.

### 2026-06-07 — Dedicated NIC architecture migration
Migrated from router-on-a-stick (single NIC, VLAN 99 WAN transit) to dedicated NICs:
- enp0s31f6 → vmbr0 (WAN bridge, direct to senate)
- enp1s0 → vmbr1 (LAN trunk, VLAN-aware, to death-star)
Eliminated VLAN 99 entirely. Switch port 1 reconfigured from senate uplink to executor
LAN trunk. tarkin hardware updated: net0 (em0) on vmbr1 no tag, net1 (vtnet0) on
vmbr0 no tag. Senator static route re-added.

### 2026-06-07 — Kea DNS misconfiguration (auto-collect option data)
**Issue:** After NIC migration, clients received `10.0.20.3` (holonet's IP) as their
DNS server. Internet routing worked but no domain resolution.
**Root cause:** Kea's "Auto collect option data" checkbox had auto-populated DNS and
router values from whatever IP it detected on the interface — holonet happened to have
gotten a lease before the correct static reservation was applied, so Kea locked onto
that IP. The incorrect values persisted in lease cache even after correction.
**Fix:** Unchecked "Auto collect option data" on all six Kea subnets. Manually set
router and DNS server fields to the correct `10.0.x.1` gateway for each subnet.
Physically disconnected and reconnected falcon's ethernet to force a fresh DHCP
discovery bypassing the cached lease.
**Learning:** Kea's auto-collect grabs whatever it finds — always configure DNS and
router options explicitly. DHCP `/release` and `/renew` is not always enough to get
a clean lease; physical disconnect or waiting for lease expiry may be required when
the server has a cached assignment for a known MAC.

---

## Key Troubleshooting Reference

| Symptom | Cause | Fix |
|---------|-------|-----|
| Kea not issuing leases | dnsmasq holding port 67 | Disable dnsmasq in Services |
| Proxmox unreachable after PVID change | Proxmox mgmt traffic in wrong VLAN | Keep trunk port PVID=1 |
| HTTPS to 10.0.10.10:8006 times out | Asymmetric routing | Add static route on vmbr1.10 |
| tarkin VLAN interfaces wrong IPs | VM NICs swapped in Proxmox | Swap bridge/tag assignments |
| NIC not coming up after hardware move | Interface name changed | Update bridge-ports in interfaces |
| WiFi SSID wrong VLAN | Switch port 3 Untagged instead of Tagged | Change to Tagged in VLAN table |
| Wrong DNS/router in DHCP lease | Kea auto-collect populated bad values | Disable auto-collect, set manually |
| /release /renew not fixing DHCP | Kea serving cached lease | Physical disconnect or wait for expiry |
| pvresize fails with duplicate UUID | Two drives with same LVM UUID | Remove old drive, retry with one drive |

---

## Skills Demonstrated

`Network segmentation` `VLAN design` `802.1Q trunking` `Firewall policy design`
`OPNsense` `Proxmox VE` `Kea DHCPv4` `Asymmetric routing diagnosis`
`Managed switch configuration` `Router-on-a-stick architecture` `Static routing`
`Linux network interfaces` `LVM administration` `NVMe migration` `dd cloning`
`Troubleshooting port conflicts` `DHCP lease management` `AI-assisted engineering workflow`
