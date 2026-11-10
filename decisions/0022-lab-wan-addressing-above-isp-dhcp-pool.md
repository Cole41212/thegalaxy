# 0022 — Lab WAN addressing sits above the ISP gateway's DHCP pool
**Status:** Accepted (2026-08-26, site relocation)

**Context:** The lab's two WAN-facing addresses — tarkin's WAN and executor's house-side
management address — are static, on a network segment the lab does not administer. At the
original site that segment was served by a consumer router, and tarkin's `192.168.1.100` was
protected by a DHCP reservation on it.

The 2026-08-26 relocation showed what that protection was actually made of. The new site's
ISP-supplied gateway serves the same `192.168.1.0/24`, so every address in the repo stayed
syntactically valid — but its DHCP pool spanned `.2–.254`, and its vendor-class (DHCP Option 60)
table had already assigned `.100` to an ISP-supplied device that provisions itself. The
reservation that had protected tarkin lived on the old router and did not travel. Two devices,
one address, one layer-2 segment: the lab came up healthy internally with no path to the
internet ([runbooks/site-relocation.md](../runbooks/site-relocation.md)).

The lab's own convention does not transfer either. Inside `10.0.0.0/16`, statics sit below
`.100` and pools run `.100–.200`, and that works because tarkin is the DHCP authority and
enforces it. On senate's segment the authority is someone else's equipment.

**Decision:** Put the lab's WAN-facing statics in a range the gateway's DHCP server
**structurally cannot assign**, instead of a range it has been asked to avoid.

- senate's DHCP pool is narrowed to `192.168.1.2–192.168.1.99`.
- tarkin's WAN moves to **`192.168.1.200`**; executor stays at `192.168.1.225`. Both are above
  the pool.
- Reservations for both are still created on the gateway as belt and braces. Being outside the
  pool is the real protection; the reservation is not relied on.
- senate's static route (`10.0.0.0/16 → 192.168.1.200`) and any port forward follow the address.

This is deliberately the inverse of the lab's internal convention, and
[network/ip-scheme.md](../network/ip-scheme.md) says so where the addresses are listed.

**Alternatives:**

*Keep `.100` and reserve it again* (rejected) — this is the arrangement that just failed. A
reservation is a request to a DHCP server the lab does not control, and the conflict came from
a mechanism (vendor-class assignment) that a reservation does not obviously override.

*Take tarkin's WAN by DHCP* (rejected) — the static route on senate and the port forward both
point at a fixed address. A lease that moves breaks house-side access to the lab silently.

*Bridge or passthrough mode on the ISP gateway* (rejected for now) — it would remove the house
segment's DHCP from the picture entirely, but it reconfigures equipment the rest of the
household depends on, for a problem that address placement already solves. Revisit if the
gateway proves unmanageable.

*Renumber the house LAN* (not needed) — only required when the gateway's LAN falls inside
`10.0.0.0/16`, which would collide with the lab supernet. It did not.

**Consequences:**

**A static address is only safe if the DHCP authority on that segment agrees it is off-limits.**
On equipment the lab does not control, the durable form of that agreement is a pool boundary,
not a reservation.

**Some of the lab's working state lives on the house router and does not move with the box** —
the pool boundary, the static route, the reservations, and any port forward. The single-box
design (ADR 0001) means relocating the lab is moving one machine, and that holds for
everything inside the box and for none of this. The relocation runbook exists to rebuild it,
in order.

**Existing leases inside the newly excluded range are not revoked.** They persist until renewal
and then migrate down on their own; nothing breaks meanwhile.

**The house segment now has two conventions in one repo** — statics above the pool on senate's
network, statics below it everywhere tarkin is the authority. The difference is intentional and
is recorded at the point of use rather than left to be rediscovered.
