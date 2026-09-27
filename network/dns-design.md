# DNS Design

**Status: Implemented 2026-07-13 (Phase 3); recursion leg broken by ISP interception
2026-09-26/27 — see the next section before relying on anything below.** SECLAB deferred to
Phase 7 as designed. Build details and incidents: phases/phase-3-dns-and-tailscale.md.

## ⚠️ Partially superseded — ISP DNS interception (2026-09-26/27)

**The recursion leg documented below does not currently work.** The ISP began transparently
intercepting all outbound port 53 traffic after a same-day plan change, which broke Unbound's
recursive resolution and nothing else. Two deviations from this document are live; both are
listed under "Live deviations" at the end of this section. Full incident narrative:
[phases/phase-6-inquisitor-wazuh.md](../phases/phase-6-inquisitor-wazuh.md).

### How it was proven
A query for a name pointed at an **RFC 5737 TEST-NET-1** address returned **NOERROR with
answers, over both UDP and TCP**. No server exists at that address and none can — the range is
reserved and unroutable — so anything that answers is not the server that was addressed. The
probe needs no baseline and no cooperation from the ISP, and is worth keeping as a general
technique for detecting silent proxying.

### Why it broke recursion and only recursion
Unbound in recursive mode asks **authoritative servers authoritative questions** and expects
non-recursive referrals, walking the delegation chain down from the root. The interceptor
answered every query **recursively**, so Unbound classified each responder **REC_LAME** and
discarded it; the delegation walk could never complete.

Every **forwarder** on the network kept working and reported healthy — Pi-hole, phones,
laptops, IoT — because a forwarder asks for a recursive answer and received exactly that. The
failure was invisible from the majority of the system, which is the opposite of the usual
"lots of things are broken, find the common cause" shape.

The onset looked gradual because it was **cache expiry**: names failed one at a time as their
TTLs aged out of Unbound's cache, not all at once when the interception began.

### Where this is going: DNS over TLS
Plaintext recursion is impossible on this connection — no Unbound configuration survives an
interceptor rewriting every port 53 answer. The resolver moves to **DoT forwarding on port
853**, which the interception does not touch. DoT was verified clean against both candidate
upstreams with **CA-chained certificates** confirmed, not merely "the connection succeeded".

**Not yet implemented, and deliberately so.** Moving from private recursion to third-party
forwarding reverses [ADR 0004](../decisions/0004-dns-topology.md), which rejected public
forwarders on privacy grounds, and reverses the "no public resolver sees the lab's query
stream" property stated under *Why this topology* above. That is a cross-cutting design
reversal, not a configuration change, so it gets its own ADR first. Per-client visibility —
the security property this whole topology exists to produce — is unaffected either way, since
it is produced at Pi-hole, upstream of this leg.

### Live deviations
| Deviation | Status | Closes when |
|---|---|---|
| Pi-hole's upstream is not Unbound-on-tarkin | Live since 2026-09-26/27 | The DoT ADR is written and implemented |
| DNSSEC validation is **off** | Live since 2026-09-26/27 | **Hard deadline: 2026-10-11 root KSK rollover** |

Recorded here rather than left as undocumented drift, because a source-of-truth document that
describes a design the network is not running is worse than one that admits the gap.

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
- Kea UI quirk (Phase 4): ordered DNS-server lists can silently re-sort on an in-place
  edit. Fix: remove all entries → re-add in order → Apply. Verify what clients actually
  received on the wire (`resolvectl status` after a renew), not in the UI.

## Firewall impact (implemented with the cutover)
FAMILY and IOTGUEST pass `→ 10.0.30.53 : 53` and `→ 10.0.30.1 : 53` (TCP/UDP) **above**
their `block → 10.0.0.0/16`; the old `→ 10.0.x.1 : 53` exceptions were deleted after
cutover verification. Phase 4 added one more exception in the same shape — pass TCP
`→ 10.0.30.50 : 8096` (Jellyfin on cantina) — so the inter-VLAN holes remain
one-host-one-port pinholes: DNS + Jellyfin, nothing broader. SECLAB has
no exception and therefore no working DNS by default — deliberate, finalized in Phase 7
(see network/firewall-rules.md).

## As implemented (Phase 3 details)
- Pi-hole listens on all interfaces / permits all origins — WHO may reach :53 is enforced
  at tarkin's firewall, the correct layer; order66 has zero inbound exposure beyond it.
- Conditional forwarding: 10.0.0.0/16 → 10.0.30.1; local domain `galaxy.internal`
  (`.internal` is the ICANN-reserved private-use TLD).
- Naming: Kea reservations for devices worth naming (cantina added in Phase 4, vault in
  Phase 5 — same DHCP-in-OS deviation, see network/ip-scheme.md). The
  os-kea-unbound plugin was declined (unsigned package on the firewall — supply chain).
  Phase 4 corrected the Phase 3 model of *how* registration happens — see the
  dependency below.
- Name-registration dependency (Phase 4 — guardrail): registration requires Unbound →
  General → **Register ISC DHCP4 Leases** and **Register DHCP Static Mappings** both
  enabled. Despite the ISC naming, on OPNsense 26.x these consume Kea's lease data.
  With them off, a reservation plus an Unbound restart registers nothing.
- Remote devices resolve through Pi-hole too: tailnet DNS override 10.0.30.53 primary /
  10.0.30.1 secondary via phantom's subnet route (ADR 0011).

## Exception — executor resolves outside the lab (Phase 6, 2026-08-17)

**executor's resolver is 192.168.1.1 (senate), not order66.** This is a stated decision, not
drift, and it is the only host in the lab deliberately outside the resolution path above.

**Why.** postfix on executor must resolve `smtp.gmail.com` before it can deliver any alert
mail. Pointing the hypervisor's resolver at a guest VM would mean **order66 going down could
suppress the alert saying order66 is down** — the notification path would run through the thing
it is reporting on. That is the same structural rule as ADR 0009's host storage leg (host↔NAS
traffic must not transit the router VM) and the same rule that keeps the dead-man's switch on an
external service (ADR 0021). No alert path may share a failure domain with what it monitors.

**Single nameserver, deliberately.** No secondary is configured. Phase 3's headline incident was
exactly a fallback masking a dead primary — Pi-hole had zero working upstreams, every client
silently used the DHCP-provided secondary, and the network stayed fully functional so nothing
surfaced it. A resolver failure on executor should be loud, and a fallback is what makes it
quiet.

**Consequences, both accepted:**

- **executor cannot resolve `.galaxy.internal`** — senate knows nothing about the private zone.
  Its Wazuh agent is therefore configured with the manager's **IP address** rather than its
  FQDN (ADR 0020).
- **executor's DNS queries do not appear in Pi-hole**, so the per-client visibility this design
  exists to provide has a hole in it precisely where the hypervisor is. Host DNS visibility
  comes from **journald ingestion** via the Wazuh agent instead — a different source answering
  the same question. Recorded in [docs/telemetry-logging.md](../docs/telemetry-logging.md).

## Build order (Phase 3 — executed 2026-07-13)
1. Build order66; install Pi-hole; set upstream = 10.0.30.1 (Unbound).
2. Switch Unbound on tarkin to recursive mode.
3. Add FAMILY/IOTGUEST firewall exceptions to 10.0.30.53.
4. Update Kea DNS options (primary .53, secondary .1) per subnet; renew leases.
5. Verify per-client logging in Pi-hole and that blocklists resolve.

All five steps done. Post-cutover, an order66-down failover drill passed on 2026-07-14 —
