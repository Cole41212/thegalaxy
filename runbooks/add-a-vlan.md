# Runbook — Add a VLAN (full cross-stack)

Order matters — firewall/DHCP before the switch port goes live.

1. **OPNsense (tarkin):** Interfaces → Devices → VLAN → add (parent = LAN/em0,
   tag = N). Assign the new interface, enable it, set static IP 10.0.N.1/24.
   (Menu is "Devices" since OPNsense 24.7 — older docs say "Other Types".)
2. **Kea DHCP:** add subnet 10.0.N.0/24, pool .100–.200. **Auto-collect OFF.** Set router
   and DNS manually (router 10.0.N.1; DNS primary 10.0.30.53, secondary 10.0.30.1 —
   order66 is live).
3. **Firewall:** copy the matching trust-zone template. Low-trust: pass DNS exception
   ABOVE the block to 10.0.0.0/16, then pass any (internet) — without the exception the
   broad block silently kills DNS (the 2026-06-22 FAMILY/IOTGUEST outage).
4. **Switch (death-star):** add VLAN N to the VLAN table; **tag** it on the trunk port
   (executor uplink); set the endpoint's access port untagged with PVID N. Save to flash
   before applying.
5. **AP (holonet)** if wireless: map an SSID to VLAN N (must be in the AP's trunk).
6. **Verify:** a client on VLAN N gets a .100–.200 lease, correct gateway + DNS, and the
   inter-VLAN policy behaves as intended.