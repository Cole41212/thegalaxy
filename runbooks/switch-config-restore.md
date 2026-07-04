# Runbook — Switch (death-star) Config & Recovery

ZX-SWTGW215AS managed switch.

## ⚠️ The critical gotcha
SAVE the config to flash BEFORE applying network changes. The web UI disconnects the
instant you apply a management IP/VLAN change — if you didn't save first, you can't, and
you're locked out until a factory reset.

## Normal access
Web UI at 10.0.10.2 (Management). Factory default is 192.168.2.1.

## Restore from export
UI → config management → import the saved configuration file → save to flash.

## Locked out → factory reset
1. Hardware reset button (hold per manual).
2. Set a PC NIC to 192.168.2.x; browse to https://192.168.2.1; login admin / admin.
3. Re-import the saved config, OR rebuild: set mgmt IP/VLAN, then the trunk port
   (executor uplink) with PVID=1 so Proxmox management stays reachable, then per-port
   VLAN tagging. Save to flash before each apply.