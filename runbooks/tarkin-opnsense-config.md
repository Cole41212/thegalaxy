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

## Firewall Rules

Rules are configured in **Firewall → Rules** (legacy interface).
All rules use Direction: `in`, Protocol: any (unless specified), Action as noted.

### TRUSTED
| # | Action | Source | Destination | Description |
|---|--------|--------|-------------|-------------|
| 1 | Pass | TRUSTED net | any | Trusted full access |

### SERVERS
| # | Action | Source | Destination | Description |
|---|--------|--------|-------------|-------------|
| 1 | Block | SERVERS net | TRUSTED net | Servers cannot reach trusted devices |
| 2 | Pass | SERVERS net | any | Servers reach internet |

### FAMILY
| # | Action | Source | Destination | Description |
|---|--------|--------|-------------|-------------|
| 1 | Block | FAMILY net | 10.0.0.0/16 | Family cannot reach homelab |
| 2 | Pass | FAMILY net | any | Family internet access |

### IOTGUEST
| # | Action | Source | Destination | Description |
|---|--------|--------|-------------|-------------|
| 1 | Block | IOTGUEST net | 10.0.0.0/16 | IoT cannot reach homelab |
| 2 | Pass | IOTGUEST net | any | IoT internet only |

### SECLAB
| # | Action | Source | Destination | Description |
|---|--------|--------|-------------|-------------|
| 1 | Block | SECLAB net | 10.0.0.0/16 | Lab fully isolated |
| 2 | Pass | SECLAB net | any | Lab reaches internet for tools |

### MGMT
| # | Action | Source | Destination | Description |
|---|--------|--------|-------------|-------------|
| 1 | Block | MGMT net | TRUSTED net | MGMT blocked from trusted |
| 2 | Pass | MGMT net | any | MGMT devices reach internet |

### WAN
- Block private networks: **DISABLED** (tarkin's WAN faces senate LAN, not internet)
- Block bogon networks: **ENABLED**
- Pass rule: `192.168.1.0/24 → 10.0.0.0/16` (house devices can reach homelab via static route)

---

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
