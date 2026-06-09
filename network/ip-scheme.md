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
| archives | TrueNAS SCALE | 10.0.30.20 | NFS, SMB, storage |
| shipyard | Docker host | 10.0.30.25 | Portainer :9443, Crafty :8000/:8443 |
| order66 | Pi-hole | 10.0.30.53 | DNS :53, Web UI :80 |
| cantina | Jellyfin | 10.0.30.50 | Media server :8096 |
| vault | Nextcloud | 10.0.30.40 | File server :443 |
| inquisitor | Wazuh SIEM | 10.0.30.100 | SIEM dashboard :443 |

### Trusted Devices (VLAN 20)

| Hostname | Role | IP | Notes |
|----------|------|----|-------|
| falcon | Main PC | 10.0.20.10 | DHCP reservation |
| scout | Laptop | DHCP | Linux Mint Cinnamon |
| comlink | iPhone | DHCP | |

### Security Lab (VLAN 60)

| Hostname | Role | IP | Notes |
|----------|------|----|-------|
| maul | Kali Linux | 10.0.60.10 | DHCP reservation |
| rogue | Vulnerable VM | 10.0.60.100 | DHCP reservation, fully isolated |

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

## Minecraft

Server runs on `shipyard` (Docker container via Crafty Controller).

| Service | Address | Port |
|---------|---------|------|
| Minecraft Java | 10.0.30.25 | 25565 |
| Crafty web UI | https://10.0.30.25:8443 | — |

Port forward: OPNsense NAT → senate port forward → tarkin → shipyard.
World files backed up on falcon at `C:\minecraft_backup\`.
