# Hardware Inventory

Single source of truth for The Galaxy hardware — models, controllers, and drive mappings.
Other docs and the session context defer to this file.

## executor — Proxmox VE 9.1.1 host
| Component | Detail |
|---|---|
| CPU | Intel Core i7-7700T |
| RAM | 64 GB DDR4 |
| Boot/VM disk | 512 GB NVMe (Samsung 970 EVO) — Proxmox OS + primary VM disks |
| SSD #1 | 512 GB SATA SSD — VM-disk store (`ssd-vmstore`) |
| SSD #2 | 512 GB SATA SSD — inquisitor (Wazuh) data (`ssd-inquisitor`) |
| GPU | discrete GPU (not yet enumerating in Proxmox — Phase 4) + Intel HD 630 iGPU |
| NICs | 2× 2.5 GbE — `enp0s31f6` (WAN→senate), `enp1s0` (LAN trunk→death-star) |

## Storage controllers & drive mapping
SSDs + NVMe stay on the motherboard SATA controller (Proxmox-owned). All 12 HDDs sit on
two add-in controllers passed through whole to archives.

| Controller | PCI ID | IOMMU group | Owner | Drives |
|---|---|---|---|---|
| Intel Q170 SATA (AHCI) | 00:17.0 | 5 | Proxmox host | 2× 512 GB SSD |
| ASM1064 SATA | 05:00.0 | 15 | archives (passthrough) | 6× 4 TB SATA |
| Broadcom SAS3008 HBA | 06:00.0 | 16 | archives (passthrough) | 6× 4 TB SAS |
| Samsung NVMe | 07:00.0 | 17 | Proxmox host | boot / VM disk |

### HDDs (confirm serials/models against TrueNAS → Storage → Disks)
| Serial | Model | Interface | Controller | Recording |
|---|---|---|---|---|
| ZDH0XCR2 | Seagate ST4000DM005 | SATA | ASM1064 | CMR |
| ZDH0XBHB | Seagate ST4000DM005 | SATA | ASM1064 | CMR |
| WD-WMC130H9P5ZS | WD Black WD4003FZEX | SATA | ASM1064 | CMR |
| WD-WMC130H3UY5Z | WD Black WD4003FZEX | SATA | ASM1064 | CMR |
| WD-WCC7K5UAHYYP | WD Red WD40EFRX | SATA | ASM1064 | CMR |
| 46SAKK23F58D | Toshiba HDWE140 | SATA | ASM1064 | CMR |
| 5000cca03b6174a0 | HGST HUS724040ALS640 | SAS | SAS3008 | CMR |
| 5000cca05c2983ec | HGST HUS724040ALS640 | SAS | SAS3008 | CMR |
| PBHRLHVX | (confirm) | SAS | SAS3008 | CMR |
| PCGRUTJX | (confirm) | SAS | SAS3008 | CMR |
| (fill) | (confirm) | SAS | SAS3008 | — |
| (fill) | (confirm) | SAS | SAS3008 | — |

All HDDs confirmed CMR (suitable for ZFS).

## ZFS pool
Pool `holocron` (confirm) — 2× 6-drive RAIDZ2 vdevs, one pool. ~29 TiB usable; tolerates
2 simultaneous failures per vdev. TrueNAS boot/system is the 32 GB SCSI vdisk on the NVMe,
not in the pool.

## Network & endpoint hardware
| Host | Device | Notes |
|---|---|---|
| death-star | ZX-SWTGW215AS — 2.5 G 8-port managed switch | 10.0.10.2 |
| holonet | TP-Link TL-WA1201 AP | Multi-SSID (VLAN 20/40/50), 10.0.20.3 |
| senate | Asus GT-AX11000 | house router (not lab-managed) |
| falcon | i5-13600K · RTX 3060Ti · 32 GB · Win10 | workstation, 10.0.20.10 |
| scout | Linux Mint laptop | future SECLAB attack box (VLAN 60) |
| comlink | iPhone | VLAN 20 |
| Pi 5 + 6 TB USB HDD | offsite backup appliance | at relative's, reachable via Tailscale (Phase 3) |
| panel-1 / panel-2 | Asus T100T / X205T | dashboard displays (Phase 6) |