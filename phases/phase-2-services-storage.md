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
| Firewall rules migrated to Rules [new] | ⬜ | Migration assistant in OPNsense |
| LSI HBA card installed | ⬜ | For HDD passthrough to TrueNAS VM |
| shipyard VM created (ID 103) | ⬜ | Ubuntu Server, VLAN 30 |
| Docker installed on shipyard | ⬜ | |
| Portainer deployed | ⬜ | https://10.0.30.25:9443 |
| Crafty Controller deployed | ⬜ | https://10.0.30.25:8443 |
| Minecraft world migrated | ⬜ | World files at C:\minecraft_backup\ on falcon |
| Minecraft port forward configured | ⬜ | OPNsense NAT + senate forward |
| archives VM created | ⬜ | TrueNAS SCALE, HDD controller passthrough |
| ZFS pool configured | ⬜ | 6×4TB — RAIDZ2 (~16TB usable) |
| NFS/SMB shares configured | ⬜ | |
| SSD storage plan implemented | ⬜ | VM backups + dedicated inquisitor storage |
| Proxmox scheduled VM backups | ⬜ | Datacenter → Backup |

---

## Hardware Notes

### LSI HBA for TrueNAS passthrough
The 6×4TB WD HDDs need to connect through a separate HBA that can be passed through
to the TrueNAS VM as a PCIe device. Do NOT use the motherboard SATA controller if it
also hosts the NVMe boot drive — you cannot split a controller between the host and a VM.

Recommended: **LSI 9207-8i** or **LSI 9211-8i** (~$30–40 used on eBay).
These are standard IT-mode HBAs with excellent TrueNAS compatibility.

### Two 512GB SSDs
Planned use:
- One SSD: Proxmox VM backup target (Datacenter → Backup → Add storage)
- One SSD: Dedicated storage for inquisitor (Wazuh) — logs accumulate fast

Both can be added as Proxmox storage via Datacenter → Storage → Add → Directory
(format with ext4) or LVM.

### ZFS Pool Design
6×4TB drives. Two options:

| Configuration | Usable | Fault Tolerance | Performance |
|---|---|---|---|
| RAIDZ2 | ~16TB | 2 drive failures | Good sequential |
| 3× mirror pairs | ~12TB | 1 failure per pair | Better random I/O |

RAIDZ2 recommended — better fault tolerance, more usable space.

---

## shipyard VM Spec

| Setting | Value |
|---------|-------|
| VM ID | 103 |
| Hostname | shipyard |
| OS | Ubuntu Server 24.04 LTS |
| Disk | 30GB, local-lvm |
| CPU | 2 cores, host type |
| RAM | 4096MB |
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

### [DATE] — Phase 2 started

