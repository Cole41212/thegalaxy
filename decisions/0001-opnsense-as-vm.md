# 0001 — OPNsense as a Proxmox VM
**Status:** Accepted (Phase 1)
**Context:** Need an enterprise-grade firewall/router for VLAN segmentation without buying
a dedicated appliance.
**Decision:** Run OPNsense (tarkin) as a VM on executor with a dedicated WAN NIC and a LAN
trunk NIC.
**Alternatives:** Dedicated hardware firewall (cost, another box); pfSense (preference);
consumer router firmware (no real segmentation).
**Consequences:** Snapshots/backups of the firewall; one less box. Tradeoff: firewall
availability is tied to the host (see 0001-risk in threat-model SPOF). Recovery via console
+ `pfctl -d`.