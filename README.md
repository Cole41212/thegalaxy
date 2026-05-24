# The Galaxy — Network & Security Home Lab

**Cybersecurity student** (Marist University, graduating 2026) building a 
segmented homelab focused on blue team skills: network defense, SIEM, 
log analysis, and vulnerability management.

## What This Is

A production-grade home network running on enterprise-style architecture:
- OPNsense firewall with full VLAN segmentation
- Proxmox hypervisor hosting all VMs and services
- Wazuh SIEM for centralized log collection and alerting
- Isolated security lab for offensive/defensive practice
- Pi-hole DNS filtering and Tailscale remote access

## Architecture

| VLAN | Name | Subnet | Purpose |
|------|------|--------|---------|
| 10 | Management | 10.0.10.0/24 | Switches, APs, Proxmox host |
| 20 | Trusted | 10.0.20.0/24 | Personal devices |
| 30 | Servers | 10.0.30.0/24 | VMs and NAS |
| 40 | Family | 10.0.40.0/24 | Family devices |
| 50 | IoT/Guest | 10.0.50.0/24 | Smart devices, guests |
| 60 | Security Lab | 10.0.60.0/24 | Isolated attack/defense |

## Hardware

| Host | Role | Specs |
|------|------|-------|
| executor | Proxmox hypervisor | HP ProDesk 600 G3, i5-7500T, 32GB RAM |
| archives | NAS (Phase 2) | HP EliteDesk 800 G3, 2×2TB HDD |
| death-star | Core switch | 2.5G 8-port managed |
| holonet | Wireless AP | Asus RT-AC68U |

## Phases

| Phase | Focus | Status |
|-------|-------|--------|
| 1 | Core network — OPNsense, VLANs, firewall | 🔄 In Progress |
| 2 | TrueNAS storage | ⬜ Not Started |
| 3 | Pi-hole DNS + Tailscale VPN | ⬜ Not Started |
| 4 | Jellyfin media server | ⬜ Not Started |
| 5 | Nextcloud file server | ⬜ Not Started |
| 6 | Wazuh SIEM + dashboard displays | ⬜ Not Started |
| 7 | Security lab — Kali + vulnerable VM | ⬜ Not Started |
| 8 | AI tools + portfolio | ⬜ Not Started |

## Skills Demonstrated
`Network segmentation` `Firewall policy` `VLAN design` `Proxmox` 
`OPNsense` `Docker` `Wazuh SIEM` `Pi-hole` `Tailscale` `Threat detection`