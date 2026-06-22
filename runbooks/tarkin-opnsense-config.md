# Runbook: tarkin OPNsense Configuration

**VM:** tarkin (ID 100) on executor (Proxmox)  
**OPNsense version:** 26.1.6  
**WAN IP:** 192.168.1.100 (static, DHCP reservation on senate)  
**LAN bootstrap:** 192.168.100.1/24 (em0, for initial access before switch config)  
**Primary access:** https://10.0.20.1 (from TRUSTED VLAN after full config)

---

## VM Hardware Configuration

| NIC | Proxmox Device | Bridge | VLAN Tag | OPNsense Interface |
|-----|----------------|--------|----------|--------------------|
| net0 | em0 | vmbr1 | none (trunk) | LAN — all VLAN sub-interfaces |
| net1 | vtnet0 | vmbr0 | none | WAN — 192.168.1.100 |

> **Critical:** net0 (em0) must have NO Proxmox VLAN tag — it receives all VLAN tags
> from the switch trunk and OPNsense reads them via sub-interfaces. net1 (vtnet0) must
> also have NO Proxmox VLAN tag — it connects to the flat WAN bridge (vmbr0) which
> carries only untagged 192.168.1.x traffic.

---

## VLAN Interfaces

All VLAN sub-interfaces are created on em0 in OPNsense.

| Name | Device | VLAN Tag | IP | DHCP Pool |
|------|--------|----------|----|-----------|
| MGMT (opt1) | vlan01 | 10 | 10.0.10.1/24 | .100–.200 |
| TRUSTED (opt2) | vlan02 | 20 | 10.0.20.1/24 | .100–.200 |
| SERVERS (opt3) | vlan03 | 30 | 10.0.30.1/24 | .100–.200 |
| FAMILY (opt4) | vlan04 | 40 | 10.0.40.1/24 | .100–.200 |
| IOTGUEST (opt5) | vlan05 | 50 | 10.0.50.1/24 | .100–.200 |
| SECLAB (opt6) | vlan06 | 60 | 10.0.60.1/24 | .100–.200 |

---

## Kea DHCPv4 Configuration

**Prerequisites:** Disable dnsmasq first (Services → Dnsmasq DNS & DHCP → uncheck Enable).
dnsmasq holds port 67 and silently blocks Kea from binding.

**Settings tab:**
- Enabled: ✅
- Interfaces: FAMILY, IOTGUEST, MGMT, SECLAB, SERVERS, TRUSTED
- Valid lifetime: 86400
- Firewall rules: ✅

**Subnets tab** (one entry per VLAN):

| Subnet | Description | Pools |
|--------|-------------|-------|
| 10.0.10.0/24 | mgmt | 10.0.10.100-10.0.10.200 |
| 10.0.20.0/24 | trusted | 10.0.20.100-10.0.20.200 |
| 10.0.30.0/24 | servers | 10.0.30.100-10.0.30.200 |
| 10.0.40.0/24 | family | 10.0.40.100-10.0.40.200 |
| 10.0.50.0/24 | iotguest | 10.0.50.100-10.0.50.200 |
| 10.0.60.0/24 | seclab | 10.0.60.100-10.0.60.200 |

---

## Firewall Rules (Rules [new] — post Phase 2)

Rules are evaluated top-down, first match wins. Order within each interface matters.

### TRUSTED (VLAN 20)
| Action | Source | Destination | Port | Notes |
|--------|--------|-------------|------|-------|
| pass | TRUSTED net | any | any | Full internet + homelab access |

### SERVERS (VLAN 30)
| Action | Source | Destination | Port | Notes |
|--------|--------|-------------|------|-------|
| block | SERVERS net | TRUSTED net | any | Servers cannot reach client devices |
| pass | SERVERS net | any | any | Internet access allowed |

### FAMILY (VLAN 40)
| Action | Source | Destination | Port | Notes |
|--------|--------|-------------|------|-------|
| pass | FAMILY net | 10.0.40.1 | 53 | Allow DNS to gateway — must be above block |
| block | FAMILY net | 10.0.0.0/16 | any | Block all homelab access |
| pass | FAMILY net | any | any | Internet access allowed |

### IOTGUEST (VLAN 50)
| Action | Source | Destination | Port | Notes |
|--------|--------|-------------|------|-------|
| pass | IOTGUEST net | 10.0.50.1 | 53 | Allow DNS to gateway — must be above block |
| block | IOTGUEST net | 10.0.0.0/16 | any | Block all homelab access |
| pass | IOTGUEST net | any | any | Internet access allowed |

### SECLAB (VLAN 60)
| Action | Source | Destination | Port | Notes |
|--------|--------|-------------|------|-------|
| block | SECLAB net | 10.0.0.0/16 | any | Fully isolated from homelab |
| pass | SECLAB net | any | any | Internet access allowed |

### MGMT (VLAN 10)
| Action | Source | Destination | Port | Notes |
|--------|--------|-------------|------|-------|
| block | MGMT net | TRUSTED net | any | Management cannot reach client devices |
| pass | MGMT net | any | any | Internet + homelab infrastructure access |

### WAN
| Action | Source | Destination | Port | Notes |
|--------|--------|-------------|------|-------|
| pass | 192.168.1.0/24 | 10.0.0.0/16 | any | Senate LAN → homelab (static route traffic) |

### NAT (Destination NAT)
| Interface | Destination | Port | Target | Notes |
|-----------|-------------|------|--------|-------|
| WAN | WAN address | 25565 | 10.0.30.25:25565 | Minecraft → shipyard/Crafty |

### Design Notes
- FAMILY and IOTGUEST DNS fix: the broad 10.0.0.0/16 block was silently dropping DNS
  queries to the VLAN gateway. Explicit port 53 pass rules added above each block rule.
- WAN block private networks is DISABLED — tarkin's WAN faces senate (192.168.1.0/24),
  not the internet directly, so blocking RFC1918 would cut off house network access.
- SECLAB has no DNS exception unlike FAMILY/IOTGUEST — lab VMs use static IPs and
  the isolation is intentional and total.

## Emergency Access

If the OPNsense web UI is unreachable:

1. Access tarkin console via Proxmox → VM 100 → Console
2. Option 8 (Shell) for command line access
3. `pfctl -d` — temporarily disables firewall (auto-restores on reboot)
4. Access web UI at `192.168.1.100` from a house network device

---

## Useful Diagnostic Commands (tarkin shell)

```bash
# Check interface states
ifconfig

# Check routing table
netstat -rn

# Check DHCP port binding
sockstat -4 -l | grep :67

# Ping test
ping -c 4 8.8.8.8

# Check firewall state table
pfctl -ss | head -20

# Reload all services
/usr/local/sbin/opnsense-service reload all
```
