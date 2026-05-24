# Phase 1 — Core Network

**Goal:** Replace flat home network with fully segmented VLAN architecture 
behind an OPNsense firewall running as a VM on Proxmox.

**Status:** 🔄 In Progress

---

## Decisions & Rationale

**Why OPNsense as a VM, not dedicated hardware?**  
Runs on existing Proxmox host (executor), saves hardware cost, 
snapshot/restore capability for safe config testing.

**Why VLAN 60 fully isolated?**  
Security lab traffic (Kali attacking vulnerable VMs) must never 
reach real network — isolation is enforced at the firewall level.

**Why static route on senate instead of double-NAT?**  
Keeps 10.0.0.0/16 traffic routable from the home network without 
hairpinning everything through two NAT layers.

---

## Checklist

| Step | Status | Notes |
|------|--------|-------|
| Proxmox VLAN-aware bridge enabled | ⬜ | |
| tarkin VM created, OPNsense installed | ⬜ | |
| OPNsense VLANs configured | ⬜ | |
| DHCP configured per VLAN | ⬜ | |
| Firewall rules configured | ⬜ | |
| death-star switch VLANs configured | ⬜ | |
| senate static route added | ⬜ | |
| Old Minecraft VM reservation set | ⬜ | |
| shipyard VM created, Docker + Portainer up | ⬜ | |
| Crafty deployed, Minecraft migrated | ⬜ | |
| Old Minecraft VM decommissioned | ⬜ | |
| All devices tested on correct VLANs | ⬜ | |

---

## Log


### [DATE]
-5/24/2026