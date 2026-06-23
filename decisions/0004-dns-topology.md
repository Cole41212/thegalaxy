# 0004 — DNS: Pi-hole VM + recursive Unbound, clients point at Pi-hole
**Status:** Accepted (Phase 3)
**Context:** Want ad/malware filtering, private recursion, and per-client query visibility
for the future SIEM.
**Decision:** clients → order66 (Pi-hole, 10.0.30.53) → Unbound on tarkin (recursive) →
roots. Kea sets primary DNS .53, secondary .1. order66 stays a VM.
**Alternatives:** Clients on .1 with Unbound forwarding to Pi-hole (loses per-client
visibility); Pi-hole on the Pi5 (unreachable if executor/host is down — Pi5 reserved for
offsite backup); public forwarders (privacy).
**Consequences:** Per-client DNS logs feed detection. Adds one firewall hole per low-trust
VLAN to 10.0.30.53:53. order66 is a dependency — hence the .1 secondary fallback.