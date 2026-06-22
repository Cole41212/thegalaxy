# Phase 2 — Services & Storage

**Goal:** Deploy core homelab services on `executor` — Docker host, Minecraft server,
TrueNAS storage VM with HDD passthrough, and firewall rules migration. Establish the
server infrastructure that all future phases depend on.

**Status:** 🔄 In Progress

---

## Decisions & Rationale

### Why Docker on a dedicated Ubuntu VM rather than Proxmox LXC?

Running Docker inside an LXC container places containers one layer from the Proxmox host
kernel. If a container is compromised, an attacker has a shorter path to the hypervisor.
A full Ubuntu VM creates a complete hardware isolation boundary — a compromised container
stays trapped in the VM. For a lab explicitly designed around security practice, correct
isolation is the point. The RAM overhead (~512MB for Ubuntu base) is negligible on 64GB.

### Why Crafty Controller for Minecraft?

Crafty is a web-based Minecraft server management panel that runs as a Docker container.
It handles server start/stop, console access, player management, scheduled backups, and
multiple server instances from a browser UI. This is cleaner than a raw Java process
and provides a management interface that makes sense in a homelab context.

### Why TrueNAS as a VM with controller passthrough rather than native Proxmox storage?

ZFS — the filesystem TrueNAS uses — requires direct hardware ownership of drives for
reliable operation. ZFS needs to issue cache flush commands, monitor SMART data, and
manage drive firmware directly. Passing individual drives as virtual disks through
Proxmox's virtualization layer breaks this. Passing through the entire HBA/SATA
controller gives TrueNAS the same direct hardware access it would have on bare metal,
while keeping it isolated as a VM.

A separate HBA card (LSI 9207-8i or similar) is required so the SATA controller for
the 6 HDDs can be passed through independently without also passing through the NVMe
boot controller.

---

## Checklist

| Step | Status | Notes |
|------|--------|-------|
| Firewall rules migrated to Rules [new] | ✅ | Migration assistant in OPNsense |
| LSI HBA card installed | ⬜ | For HDD passthrough to TrueNAS VM |
| shipyard VM created (ID 103) | ✅ | Ubuntu Server, VLAN 30 |
| Docker installed on shipyard | ✅ | |
| Portainer deployed | ✅ | https://10.0.30.25:9443 |
| Crafty Controller deployed | ✅ | https://10.0.30.25:8443 |
| Minecraft world migrated |✅ | World files at C:\minecraft_backup\ on falcon |
| Minecraft port forward configured | ✅ | OPNsense NAT + senate forward |
| archives VM created | ⬜ | TrueNAS SCALE, HDD controller passthrough |
| ZFS pool configured | ⬜ | 8×4TB — RAIDZ2 (~24TB usable) |
| NFS/SMB shares configured | ⬜ | |
| SSD storage plan implemented | ✅ | VM backups + dedicated inquisitor storage |
| Proxmox scheduled VM backups | ✅ | Datacenter → Backup |

---

## Hardware Notes

### Storage Layout

| Drive | Role |
|---|---|
| 512GB NVMe (Samsung 970 EVO) | Proxmox OS + all VM system disks (465GB usable) |
| 512GB SSD #1 | VM backup target (Proxmox Datacenter → Backup) |
| 512GB SSD #2 | inquisitor (Wazuh) dedicated log/data disk |
| 8× 4TB HDD → TrueNAS | RAIDZ2 pool (~24TB usable) — media, files, bulk data |

NVMe constraint: keep VM system disks lean. VMs that store bulk data (cantina, vault,
archives) keep only OS + application on NVMe; actual data lives on TrueNAS via NFS.

### LSI HBA for TrueNAS passthrough
The 8×4TB WD HDDs need to connect through a separate HBA passed through to the TrueNAS
VM as a PCIe device. Do NOT use the motherboard SATA controller if it also hosts the
NVMe boot drive — you cannot split a controller between the host and a VM.

Recommended: LSI 9207-8i or LSI 9211-8i (~$30–40 used on eBay).
These are standard IT-mode HBAs with excellent TrueNAS compatibility.

### ZFS Pool Design
8×4TB drives in RAIDZ2: ~24TB usable, tolerates any 2 simultaneous drive failures.
---

## shipyard VM Spec

| Setting | Value |
|---------|-------|
| VM ID | 103 |
| Hostname | shipyard |
| OS | Ubuntu Server 24.04 LTS |
| Disk | 30GB, local-lvm |
| CPU | 2 cores, host type |
| RAM | 8192MB |
| Network | vmbr1, VLAN tag 30, VirtIO |
| IP | 10.0.30.25 (DHCP reservation) |

Services:
- Portainer CE: `https://10.0.30.25:9443`
- Crafty Controller: `https://10.0.30.25:8443`
- Minecraft Java: `10.0.30.25:25565`

---

## archives VM Spec

| Setting | Value |
|---------|-------|
| Hostname | archives |
| OS | TrueNAS SCALE |
| Boot disk | 16GB minimum, local-lvm |
| CPU | 2 cores, host type |
| RAM | 8192MB (ZFS ARC cache) |
| Network | vmbr1, VLAN tag 30, VirtIO |
| Storage | HBA PCIe passthrough |
| IP | 10.0.30.20 (DHCP reservation) |

---

## Log

### 2026-06-09 — Phase 2 started

### 2026-06-11 — Phase 2 partial: firewall migration + shipyard + Crafty

Migrated OPNsense firewall rules from legacy interface to Rules [new] using the
built-in migration assistant. Verified all 6 interface rulesets carried over correctly.
Legacy rules disabled.

Created shipyard VM (ID 103) on VLAN 30 — Ubuntu Server 24.04 LTS, static IP
10.0.30.25 via Netplan. Installed Docker via official install script. Deployed
Portainer CE (https://10.0.30.25:9443). RAM bumped to 8GB for Minecraft workload.

Deployed Crafty Controller 4.10.4 as a Docker container via Portainer with bind mounts
to /opt/crafty/{servers,config,logs,backups}. Migrated existing Paper 1.21.1 Minecraft
world via Crafty's zip import. Server running locally on 10.0.30.25:25565.

Corrected hardware documentation: executor has 512GB NVMe (not 1TB), 2× spare 512GB
SSDs, and 8× 4TB HDDs (not 6). Storage plan revised accordingly.

### 2026-06-22 — Minecraft port forward, firewall fixes, SSD storage, scheduled backups

Configured Minecraft port forward for external access. Added Destination NAT rule on
tarkin (OPNsense) forwarding WAN:25565 → 10.0.30.25:25565 with associated filter rule.
Added matching port forward on senate (Asus GT-AX11000) forwarding external 25565 →
192.168.1.100:25565. Verified external connectivity — server reachable via public IP.

Fixed FAMILY and IOTGUEST firewall rules. Both VLANs had a broad block rule covering
10.0.0.0/16 which was also blocking DNS queries to the VLAN gateway (10.0.40.1 and
10.0.50.1 respectively). Added explicit pass rules above each block rule allowing TCP/UDP
port 53 to the gateway. Internet access and app functionality restored on both networks.

Registered two 512GB SSDs as Proxmox storage targets. Both drives had existing NTFS
partitions from prior use — wiped via pve → Disks → Wipe Disk before formatting.
Used pve → Disks → Directory → Create (ext4) for both:
- /dev/sdc (Samsung 860 EVO) → ssd-backups — content: Backup + general
- /dev/sdd (Crucial BX200) → ssd-inquisitor — content: Disk image (reserved for Wazuh)

Configured scheduled VM backups: Datacenter → Backup → nightly 2:00, all VMs, ssd-backups
target, snapshot mode, zstd compression, 7-backup rolling retention.