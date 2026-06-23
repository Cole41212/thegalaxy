# 0002 — Dedicated NICs over router-on-a-stick
**Status:** Accepted (Phase 1, superseded the initial RoaS build)
**Context:** Initial build carried WAN + LAN on one NIC via a VLAN 99 transit — fragile and
hard to reason about.
**Decision:** Two physical NICs — enp0s31f6 = WAN to senate, enp1s0 = LAN trunk to
death-star. VLAN 99 eliminated; switch sits fully behind the firewall.
**Alternatives:** Keep RoaS (simpler cabling, more fragile); LAGG (overkill here).
**Consequences:** Clean WAN/LAN separation matching enterprise edge design. Requires 2 NICs.