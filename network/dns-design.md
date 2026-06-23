# DNS Design

## Goal
One network-wide filtering resolver with private recursion, and per-client query
visibility for security monitoring.

## Resolution path
    client → order66 (Pi-hole, 10.0.30.53) → Unbound on tarkin (recursive) → root servers

- **order66 (Pi-hole, VM 104, 10.0.30.53):** the DNS server every client uses. Blocklists
  (ads/trackers/malware) plus — critically — per-client query logs.
- **Unbound on tarkin (OPNsense):** Pi-hole's single upstream, in *recursive* mode (resolves
  via the root/TLD servers directly), so no public resolver sees the lab's query stream.

## Why this topology
- **Per-client visibility is a security feature.** Clients hit Pi-hole directly, so Pi-hole
  records *which host* asked for a domain — exactly the signal the future SIEM (inquisitor)
  will correlate. Hiding clients behind tarkin would collapse every query to "from tarkin"
  and destroy that signal.
- **Recursive Unbound** keeps resolution private and drops dependence on 1.1.1.1 / 8.8.8.8.
- order66 stays a **VM**, not the Pi5: tarkin (router/DHCP) is itself a VM on executor, so
  if executor is down, routing and DHCP are already gone — a standalone Pi-hole would be
  unreachable anyway. The Pi5 is reserved for offsite backup.

## DHCP (Kea on tarkin)
- Per-subnet **primary DNS = 10.0.30.53** (auto-collect stays OFF; set manually — guardrail).
- Per-subnet **secondary DNS = 10.0.30.1** (tarkin/Unbound) so resolution survives an
  order66 outage. Without a fallback, a Pi-hole reboot takes DNS down lab-wide.
- DNS-option changes only apply on lease renewal — force a renew or bounce the client.

## Firewall impact (changes with the DNS cutover)
Today FAMILY and IOTGUEST allow `→ 10.0.x.1 : 53` above their `block → 10.0.0.0/16`. Once
clients point at 10.0.30.53 that exception stops matching and DNS breaks for those VLANs.
Replace it on FAMILY and IOTGUEST:
- `pass [VLAN] net → 10.0.30.53 : 53 (TCP/UDP)` **above** the 10.0.0.0/16 block.
- Optional fallback: also allow `→ 10.0.30.1 : 53`.
This is the only new hole — one host, port 53. SECLAB keeps gateway DNS or none (Phase 7).

## Build order (Phase 3)
1. Build order66; install Pi-hole; set upstream = 10.0.30.1 (Unbound).
2. Switch Unbound on tarkin to recursive mode.
3. Add FAMILY/IOTGUEST firewall exceptions to 10.0.30.53.
4. Update Kea DNS options (primary .53, secondary .1) per subnet; renew leases.
5. Verify per-client logging in Pi-hole and that blocklists resolve.