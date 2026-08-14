# Hardware Inventory

Single source of truth for The Galaxy hardware — models, controllers, and drive mappings.
Other docs and the session context defer to this file.

## executor — Proxmox VE 9.1.1 host
| Component | Detail |
|---|---|
| CPU | Intel Core i7-7700T |
| RAM | 64 GB DDR4 |
| Boot/VM disk | 512 GB NVMe (Samsung 970 EVO) — Proxmox OS + primary VM disks |
| SSD #1 | 512 GB SATA SSD — `ssd-vmstore`: VM disks + infrastructure VM backups |
| SSD #2 | 512 GB SATA SSD — inquisitor (Wazuh) data (`ssd-inquisitor`) |
| GPU | XFX RX 5700 DD Ultra 8 GB — passed through to cantina (VM 106); Intel HD 630 iGPU — host console. Detail: GPU section below |
| NICs | 2× 2.5 GbE — `enp0s31f6` (onboard, WAN→senate), `enp5s0` (RTL8125 @ 05:00.0, LAN trunk→death-star) |

### Host network
| Interface | Role | Address | Notes |
|---|---|---|---|
| enp0s31f6 → vmbr0 | WAN bridge | 192.168.1.225/24 | tarkin WAN only; not a VM data path |
| enp5s0 → vmbr1 | LAN trunk | — | VLAN-aware (802.1Q), `bridge-vids 2-4094` |
| vmbr1.10 | host management | 10.0.10.10/24 | + route `10.0.0.0/16 via 10.0.10.1` |
| vmbr1.30 | host storage leg | 10.0.30.2/24 | no gateway — host↔archives NFS stays on the bridge, independent of tarkin (ADR 0009) |

NIC naming: the LAN trunk is `enp5s0` — renamed from `enp3s0` when the GPU began
enumerating (Phase 4), and mis-documented as `enp1s0` before that. Predictable NIC names
encode PCI topology and move with it; re-verify names (and hostpci addresses) after any
hardware change.

### GPU — XFX RX 5700 DD Ultra 8 GB (→ cantina)
XFX RX 5700 DD Ultra 8 GB (board RX-57XL8L V1.2) — Navi 10, non-XT (36 active CUs),
VBIOS `113-1702NAVI10XL8GD6_MS_190612_W8`. Requires 8-pin + 6-pin aux power — unplugged
aux leads were the root cause of the card's original non-enumeration. No FLR; D3cold
disabled via udev (runbooks/pcie-passthrough.md, GPU addendum). PCIe atomics are
unavailable through QEMU — irrelevant to VA-API transcode, relevant only to any future
ROCm use.

Four PCI functions, all in IOMMU group 2 (the card carries an on-die PCIe switch):

| PCI addr | Function | Device ID | Disposition |
|---|---|---|---|
| 01:00.0 | PCIe upstream port | 1002:1478 | host (`pcieport`) — bridge, never passed |
| 02:00.0 | PCIe downstream port | 1002:1479 | host (`pcieport`) — bridge, never passed |
| 03:00.0 | GPU | 1002:731f (subsys 1682:5705) | → cantina (VM 106) |
| 03:00.1 | HDMI audio | 1002:ab38 | → cantina (VM 106) |

## Controllers & drive mapping
SSDs + NVMe stay on the motherboard SATA controller (Proxmox-owned). All 12 HDDs sit on
two add-in controllers passed through whole to archives.

| Controller | PCI ID | IOMMU group | Owner | Drives |
|---|---|---|---|---|
| Intel Q170 SATA (AHCI) | 00:17.0 | 5 | Proxmox host | 2× 512 GB SSD |
| RTL8125 2.5 GbE NIC | 05:00.0 | — | Proxmox host | — (LAN trunk, `enp5s0`) |
| ASM1064 SATA | 07:00.0 | 15 | archives (passthrough) | 6× 4 TB SATA |
| Broadcom SAS3008 HBA | 08:00.0 | 16 | archives (passthrough) | 6× 4 TB SAS |
| Samsung NVMe | 09:00.0 | 17 | Proxmox host | boot / VM disk |

Addresses are post-Phase 4: when the GPU began enumerating (buses 01–03), everything
behind it shifted — ASM1064 / SAS3008 / NVMe moved from 05/06/07 to 07/08/09, and
archives' hostpci entries were corrected to match. PCI addresses encode topology;
re-verify them (and NIC names) after any hardware change.

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
| PEJSRYWX | HGST HUS72404CLAR4000 | SAS | SAS3008 | CMR |
| (fill) | (confirm) | SAS | SAS3008 | — |

All HDDs confirmed CMR (suitable for ZFS).

**2026-08-11 — SAS drive replacement.** Serial PBG7ZLHX (HGST HUS724040ALS640) FAULTED with
unrecovered read errors surfaced by the weekly scrub, and was replaced by PEJSRYWX
(HGST HUS72404CLAR4000 — the EMC-branded variant of the same Ultrastar 7K4000). Cold-swapped
into the same physical slot, where it took over the device name `sdd`; the pool resilvered
with 0 errors. Logical block size was verified as 512 bytes with `sg_readcap --long`
**before** installing — ex-array enterprise SAS drives, EMC-branded ones especially, are
frequently 520-byte formatted and unusable by ZFS without a multi-hour `sg_format`. Full
write-up in [phases/phase-5-vault-immich.md](../phases/phase-5-vault-immich.md).

PBG7ZLHX was never individually mapped to a row in the table above, so the removed drive
cannot be pinned to a specific line — and two models plus one serial in the SAS group are
still unconfirmed. Re-confirm the whole SAS group against TrueNAS → Storage → Disks at the
next opportunity.

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
| echo-base | Raspberry Pi 5 · Ubuntu 24.04 arm64 · USB-C 4-bay enclosure (ASMedia ASM235CM bridge, VIA Labs hub, UAS) · 1× 6 TB SATA seated | offsite backup appliance, operational 2026-08-14 — separate physical site, ethernet + DHCP, Tailscale-only (ADR 0010). Pool `carbonite`, encrypted, single-disk by choice. 3 spare drives (4/3/1 TB) held cold |
| panel-1 / panel-2 | Asus T100T / X205T | dashboard displays (Phase 6) |