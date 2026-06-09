# Runbook: death-star Switch VLAN Configuration

**Device:** ZX-SWTGW215AS (2G05110GSM) 2.5G 8-port managed switch
**Management IP:** 10.0.10.2 (after configuration)
**Default IP:** 192.168.2.1 (factory default)
**Credentials:** admin / [your password]

Use this runbook to configure or reconfigure death-star from scratch.
This reflects the final dedicated-NIC architecture — executor's LAN NIC (enp1s0)
connects to port 1, and the Asus cable connects directly to executor's WAN NIC
(not the switch). There are three active ports.

---

## Active Port Layout

| Port | Device | Role |
|------|--------|------|
| 1 | executor (enp1s0, LAN NIC) | VLAN trunk — carries all homelab VLANs |
| 2 | falcon | Trusted workstation |
| 3 | holonet | WiFi AP — three SSIDs across three VLANs |
| 4–8 | unused | Available for future devices |

---

## Prerequisites

- Physical access to the switch
- A device with ethernet (falcon or scout)
- executor and tarkin already running

---

## Step 1 — Initial Access

If switch is at factory default, set your PC to a static IP:
- IP: `192.168.2.50`
- Subnet: `255.255.255.0`
- Gateway: `192.168.2.1`

Open `http://192.168.2.1`. Login: `admin` / `admin`.

If already configured at `10.0.10.2`, access from falcon on VLAN 20.

---

## Step 2 — Create VLANs

**Configuration → VLAN → 802.1Q VLAN**

Create each VLAN — enter ID and name, set port radio buttons, click **Add/Modify**.

> NM = Not Member

**VLAN 10 — mgmt**

| P1 | P2 | P3 | P4–8 |
|----|----|----|----|
| Tagged | NM | NM | NM |

**VLAN 20 — trusted**

| P1 | P2 | P3 | P4–8 |
|----|----|----|----|
| Tagged | Untagged | Tagged | NM |

**VLAN 30 — servers**

| P1 | P2 | P3 | P4–8 |
|----|----|----|----|
| Tagged | NM | NM | NM |

**VLAN 40 — family**

| P1 | P2 | P3 | P4–8 |
|----|----|----|----|
| Tagged | NM | Tagged | NM |

**VLAN 50 — iotguest**

| P1 | P2 | P3 | P4–8 |
|----|----|----|----|
| Tagged | NM | Tagged | NM |

**VLAN 60 — seclab**

| P1 | P2 | P3 | P4–8 |
|----|----|----|----|
| Tagged | NM | NM | NM |

> VLAN 99 does NOT exist in this architecture — do not create it.

---

## Step 3 — Set PVIDs

**Configuration → VLAN → 802.1Q VID**

| Port | PVID | Reason |
|------|------|--------|
| 1 | 1 | Trunk — all VM traffic pre-tagged, PVID irrelevant |
| 2 | 20 | falcon — untagged frames → Trusted |
| 3 | 20 | holonet — AP management → Trusted |
| 4–8 | 1 | Unused (default) |

---

## Step 4 — Change Management IP

**System → IP Setting**
- DHCP: Disable
- IP: `10.0.10.2`
- Netmask: `255.255.255.0`
- Gateway: `10.0.10.1`
- Apply

Switch UI becomes unreachable at 192.168.2.1 — expected.

---

## Step 5 — Save to Flash

**Tools → Save → Save**

---

## Step 6 — Restore Workstation Network

Set workstation ethernet back to automatic DHCP.
After ~15 seconds you should receive `10.0.20.x`.

Verify:
- `https://10.0.10.10:8006` — Proxmox
- `http://10.0.10.2` — death-star
- `https://10.0.20.1` — tarkin

---

## Adding a New Wired Device

To add a device on VLAN 30 (servers) to port 4:

1. **Configuration → VLAN → 802.1Q VLAN** → click VLAN 30
2. Set port 4 to Untagged → Add/Modify
3. **Configuration → VLAN → 802.1Q VID** → port 4 → PVID `30` → Apply
4. **Tools → Save → Save**

Same pattern for any VLAN — set the port as Untagged in the correct VLAN,
set PVID to match, save to flash.

---

## Factory Reset

Hold reset button on switch for 10 seconds until LEDs flash.
Switch reverts to 192.168.2.1. Begin from Step 1.
