# DNS Design

**Status: Implemented 2026-07-13 (Phase 3).** SECLAB deferred to Phase 7 as designed.
Build details and incidents: phases/phase-3-dns-and-tailscale.md.

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

## Firewall impact (implemented with the cutover)
FAMILY and IOTGUEST pass `→ 10.0.30.53 : 53` and `→ 10.0.30.1 : 53` (TCP/UDP) **above**
their `block → 10.0.0.0/16`; the old `→ 10.0.x.1 : 53` exceptions were deleted after
cutover verification. This is the only inter-VLAN hole — two hosts, one port. SECLAB has
no exception and therefore no working DNS by default — deliberate, finalized in Phase 7
(see network/firewall-rules.md).

## As implemented (Phase 3 details)
- Pi-hole listens on all interfaces / permits all origins — WHO may reach :53 is enforced
  at tarkin's firewall, the correct layer; order66 has zero inbound exposure beyond it.
- Conditional forwarding: 10.0.0.0/16 → 10.0.30.1; local domain `galaxy.internal`
  (`.internal` is the ICANN-reserved private-use TLD).
- Naming: Kea reservations for devices worth naming — Kea registers only static
  reservations in Unbound (on an Unbound restart), never dynamic lease hostnames. The
  os-kea-unbound plugin was declined (unsigned package on the firewall — supply chain).
- Remote devices resolve through Pi-hole too: tailnet DNS override 10.0.30.53 primary /
  10.0.30.1 secondary via phantom's subnet route (ADR 0011).

## Build order (Phase 3 — executed 2026-07-13)
1. Build order66; install Pi-hole; set upstream = 10.0.30.1 (Unbound).
2. Switch Unbound on tarkin to recursive mode.
3. Add FAMILY/IOTGUEST firewall exceptions to 10.0.30.53.
4. Update Kea DNS options (primary .53, secondary .1) per subnet; renew leases.
5. Verify per-client logging in Pi-hole and that blocklists resolve.

All five steps done. Post-cutover, an order66-down failover drill passed on 2026-07-14 —
genuinely, after the zero-upstream incident was fixed (see the Phase 3 log).