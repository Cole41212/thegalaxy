# Runbook: executor Proxmox Network Configuration

**Host:** executor (Proxmox VE 9.1.1)  
**NICs:** enp0s31f6 (onboard 2.5G, WAN), enp1s0 (PCIe 2.5G, LAN trunk)  
**Management:** https://10.0.10.10:8006 (from TRUSTED) or https://192.168.1.225:8006

---

## Network Architecture

Two physical NICs serve different purposes:

| NIC | Bridge | Role | Connected to |
|-----|--------|------|-------------|
| enp0s31f6 | vmbr0 | WAN — plain bridge | senate (house router) |
| enp1s0 | vmbr1 | LAN — VLAN-aware trunk | death-star switch port 1 |

`vmbr0` carries untagged traffic for:
- Proxmox host management (192.168.1.225)
- tarkin WAN NIC (vtnet0, 192.168.1.100)

`vmbr1` carries tagged VLAN traffic (IDs 10–60) for:
- tarkin LAN NIC (em0, all VLAN sub-interfaces)
- All future VMs on VLAN 30 (tag their NIC accordingly in VM hardware)

`vmbr1.10` provides Proxmox host management on VLAN 10:
- IP: 10.0.10.10/24
- Static route ensures homelab response traffic routes through tarkin, not senate

---

## /etc/network/interfaces

```
auto lo
iface lo inet loopback

iface enp0s31f6 inet manual

iface enp1s0 inet manual

# WAN bridge — dedicated NIC, direct to senate
auto vmbr0
iface vmbr0 inet static
        address 192.168.1.225/24
        gateway 192.168.1.1
        bridge-ports enp0s31f6
        bridge-stp off
        bridge-fd 0

# LAN bridge — VLAN-aware trunk to death-star
auto vmbr1
iface vmbr1 inet manual
        bridge-ports enp1s0
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094

# Proxmox management on VLAN 10
auto vmbr1.10
iface vmbr1.10 inet static
        address 10.0.10.10/24
        post-up ip route add 10.0.0.0/16 via 10.0.10.1
        pre-down ip route del 10.0.0.0/16 via 10.0.10.1

source /etc/network/interfaces.d/*
```

---

## Why the Static Route Is Required

Proxmox's default gateway is `192.168.1.1` (senate). When a device on VLAN 20
(e.g., falcon at `10.0.20.101`) connects to Proxmox at `10.0.10.10`, Proxmox
receives the connection correctly. However, when sending a response to `10.0.20.101`,
Proxmox checks its routing table — there is no route for `10.0.20.x`, so it falls back
to the default gateway (`192.168.1.1`). Senate has no route to `10.0.20.x` and drops
the response. The TCP handshake never completes.

The `post-up ip route add 10.0.0.0/16 via 10.0.10.1` line adds a static route so that
all `10.0.x.x` traffic is sent to tarkin (at `10.0.10.1`), which knows all homelab
subnets and routes correctly.

The `pre-down ip route del` line removes the route cleanly when the interface goes down,
preventing stale routes. The `post-up/pre-down` pattern is standard `ifupdown` practice.

---

## VM NIC Configuration for Servers VLAN

When creating VMs that should be on VLAN 30 (Servers):

In VM Hardware → Network Device:
- Bridge: `vmbr1`
- VLAN Tag: `30`
- Model: VirtIO

Do not use `vmbr0` for VM NICs unless you specifically want a VM on the flat house
network (only `tarkin` needs this for its WAN interface).

---

## Applying Changes

After editing `/etc/network/interfaces`:

```bash
ifreload -a
```

This applies changes without a full reboot. Verify with:

```bash
ip addr show
bridge vlan show
ip route show
```

---

## Diagnostic Commands

```bash
# Show all interfaces and IPs
ip addr show

# Show routing table
ip route show

# Show bridge VLAN membership
bridge vlan show

# Show bridge VLAN for tarkin's LAN NIC specifically
bridge vlan show dev tap100i0

# Verify static route is active
ip route show | grep 10.0.0.0

# Check which NIC is connected (has carrier)
ip link show | grep -A 1 "enp"
```
