# Phase 2 — Services & Storage

**Goal:** Deploy core homelab services on `executor` — Docker host, Minecraft server,
TrueNAS storage VM with HDD passthrough, and firewall rules migration. Establish the
server infrastructure that all future phases depend on.

**Status:** ✅ Complete

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

The 12 HDDs are split across two controllers — a PCIe SATA controller (6 drives) and
a SAS HBA (6 drives) — both separate from the motherboard SATA controller that holds
the SSDs and the M.2 NVMe boot drive. Both the SATA and SAS controllers are passed
through whole to archives, giving TrueNAS direct hardware control of all 12 drives,
while Proxmox keeps full ownership of its own drives. A controller cannot be split
between host and guest, so this physical separation is what makes the passthrough clean.

---

## Checklist

| Step | Status | Notes |
|------|--------|-------|
| Firewall rules migrated to Rules [new] | ✅ | Migration assistant in OPNsense |
| Storage controllers identified + IOMMU isolation verified | ✅ | SATA + SAS controllers |
| shipyard VM created (ID 103) | ✅ | Ubuntu Server, VLAN 30 |
| Docker installed on shipyard | ✅ | |
| Portainer deployed | ✅ | https://10.0.30.25:9443 |
| Crafty Controller deployed | ✅ | https://10.0.30.25:8443 |
| Minecraft world migrated |✅ | World files at C:\minecraft_backup\ on falcon |
| Minecraft port forward configured | ✅ | OPNsense NAT + senate forward |
| archives VM created | ✅ | TrueNAS SCALE, HDD controller passthrough |
| ZFS pool configured | ✅ | 12×4TB — 2× 6-drive RAIDZ2 (~29 TiB usable) |
| NFS/SMB shares configured | ✅ | |
| archives PCIe passthrough (both controllers) | ✅ | Raw Device + All Functions + PCIe |
| TrueNAS static IP 10.0.30.20 | ✅ | + Kea reservation |
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
| 12× 4TB HDD → TrueNAS | 2× 6-drive RAIDZ2 (~29 TiB usable) — media, files, bulk data |

NVMe constraint: keep VM system disks lean. VMs that store bulk data (cantina, vault,
archives) keep only OS + application on NVMe; actual data lives on TrueNAS via NFS.

### Storage Controller Passthrough for TrueNAS
The 12×4TB HDDs connect via two controllers, both passed through to archives:
- PCIe SATA controller — 6 drives
- SAS HBA — 6 drives

The 2×512GB SSDs and the M.2 NVMe boot drive stay on the motherboard SATA controller,
owned by Proxmox. A controller can't be shared between host and guest.

Controller note: the SAS HBA is the ideal controller for ZFS (direct drive access,
clean SMART passthrough). The PCIe SATA card is a multi-port consumer controller — it
works, but monitor SMART data and watch for dropped drives during the first full scrub.
If reliability issues appear under load, consolidating onto LSI HBAs (IT mode) is the
upgrade path.

### ZFS Pool Design
12×4TB drives as 2× 6-drive RAIDZ2 vdevs striped into one pool.
- Each vdev: 4 data + 2 parity; each tolerates 2 simultaneous failures.
- Usable: ≈32 TB raw / ~29 TiB after ZFS overhead.
- Chosen over a single 12-wide RAIDZ2 for faster resilvers (a rebuild only reads the
  affected 6-drive vdev, not all 12) and better IOPS (2 vdevs ≈ 2× the write throughput).
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
| Boot disk | 32GB, local-lvm |
| CPU | 2 cores, host type |
| RAM | 16384MB (ZFS ARC; ballooning OFF) |
| Network | vmbr1, VLAN tag 30, VirtIO |
| Storage | SATA + SAS controllers via PCIe passthrough — 12× 4TB |
| Machine | q35 (required for PCIe passthrough) |
| BIOS | OVMF (UEFI) + EFI disk |
| IP | 10.0.30.20 (DHCP reservation) |

## Storage Pool & Shares

Pool: `holocron` ← confirm — 2× 6-drive RAIDZ2 vdevs, unencrypted root.
- ≈32 TB raw / ~29 TiB usable; each vdev tolerates 2 simultaneous drive failures.
- Two 6-drive vdevs rather than one 12-wide: faster resilvers (only the affected vdev
  is read) and ~2× write throughput.

Drives: 6 SATA on the ASM1064, 6 SAS on the SAS3008 — both controllers passed through.

| Dataset | Preset | Purpose |
|---------|--------|---------|
| media | Generic | cantina / Jellyfin (Phase 4) |
| files | Generic | vault / Nextcloud (Phase 5) |
| crafty-backups | Generic | shipyard Minecraft backups |
| vm-backups | Generic | Proxmox overflow backups |

NFS shares: one per dataset, allowed network scoped to 10.0.30.0/24.
Data Protection: periodic snapshots on media + files; default weekly scrub enabled.

## Log

### 2026-06-09 — Phase 2 started

Phase 2 started.

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

### 2026-06-22 — archives TrueNAS VM + dual-controller passthrough (Phase 2 complete)

Corrected drive count to 12× 4TB: 6 SATA (2× Seagate ST4000DM005 CMR, 2× WD Black
WD4003FZEX, 1× WD Red WD40EFRX, 1× Toshiba HDWE140) on the ASM1064, and 6 SAS
(incl. 2× HGST HUS724040ALS640) on the SAS3008. All CMR.

Verified IOMMU enabled by default (PVE 9 / kernel 6.x). Identified controllers via
`pvesh get /nodes/pve/hardware/pci`: ASM1064 SATA at 05:00.0 (IOMMU group 15), SAS3008
at 06:00.0 (group 16) — each isolated, clear of the NVMe and motherboard SATA controller.

Created archives VM (ID 102): q35 + OVMF, 32GB boot disk, 2 cores host, 16GB RAM
(ballooning off), VLAN 30. Passed through both controllers via GUI
(Raw Device + All Functions + PCI-Express). Installed TrueNAS SCALE, static IP 10.0.30.20.
Built pool 'holocron' as 2× 6-drive RAIDZ2 (~29 TiB), datasets media/files/crafty-backups/
vm-backups with NFS shares scoped to 10.0.30.0/24. Excluded archives from the vzdump job
(pool protected by RAIDZ2 + ZFS snapshots + weekly scrub).

#### Issue: TrueNAS installer blocked at boot — "Access Denied" on the DVD-ROM
**Root cause:** The EFI disk was created with `pre-enrolled-keys=1`, so OVMF booted with
Secure Boot enforcing. The TrueNAS installer isn't signed with Microsoft's keys, and
TrueNAS doesn't support Secure Boot, so UEFI rejected it.
**Fix:** Removed and recreated the EFI disk with "Pre-Enroll keys" unchecked → Secure
Boot off → installer loaded.
**Learning:** For OVMF VMs running OSes that don't support Secure Boot (TrueNAS, many
appliances), create the EFI disk without pre-enrolled keys.

#### Issue: noVNC console rendered as colored static during install
**Root cause:** Cosmetic framebuffer rendering glitch when TrueNAS switches display mode
under OVMF + default display in noVNC. Installer was running fine underneath.
**Fix:** Reopened the console / forced a repaint. (Switching the Display type also works.)
**Learning:** Garbled noVNC output during a mode switch is a rendering artifact, not a
crash — repaint or change display type rather than assuming install failure.

#### Issue: only 8 of 12 drives visible in TrueNAS
**Root cause:** One of the two SFF-8643 (mini-SAS HD) breakout cables on the SAS3008 was
not fully seated. The affected SAS drives had power (all spinning) but no data path, so
they never reached the controller. Power and data are separate connections on SAS drives.
**Diagnosis:** Decoded drive serials to confirm all 6 SATA drives present and the SAS side
short; `lspci` confirmed the SAS3008 itself initialized and was presenting some drives, so
the controller and passthrough were fine — narrowed it to per-cable seating.
**Fix:** Reseated the SFF-8643 cable at the card; all 12 drives appeared.
**Learning:** A spinning drive that doesn't enumerate is almost always a data-cable, not a
power, problem. When a controller shows *some* of its drives, suspect a specific cable, not
the controller or the passthrough.