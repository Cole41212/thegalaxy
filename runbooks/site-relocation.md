# Runbook: Site Relocation

**Purpose:** Move the lab to a new physical site and restore full service. Written
after the 2026-08-26 move; the log at the bottom is the record of that one.

ADR 0001 put the firewall on the hypervisor instead of in its own appliance — "one less
box" — and the practical upshot is that *moving the lab means moving one machine*. That
held. What it did not cover — and what this runbook exists for — is that a working lab also
depends on state living on the **house router**, and none of that state moves with the box.

---

## What travels, and what doesn't

| Travels (inside the box) | Does **not** travel (lives on senate) |
|---|---|
| All six VLANs, their gateways and pools | The DHCP reservation protecting tarkin's WAN address |
| Every firewall rule and NAT entry | The static route `10.0.0.0/16 → tarkin` |
| Kea, Unbound, Pi-hole, all leases and gravity | Any port forward (e.g. TCP 25565) |
| Every VM, dataset, snapshot and mount | The DHCP pool boundaries that keep the lab's statics safe |
| Tailscale identity (key expiry off — phantom self-heals) | Whatever the new gateway self-provisions on its own |

The lab-side column is why the box comes up talking to itself perfectly while having
no path to the internet. **Internal health is not evidence of external health**, and
the two failure domains should be tested separately.

---

## Pre-move

1. Note the current senate-side state you will have to rebuild: reservation address,
   static route, port forwards.
2. Record tarkin's WAN MAC (`ifconfig vtnet0 | grep ether`) and executor's
   (`ip link show enp0s31f6`) — you will need them for reservations at the new site.
3. Clean shutdown of guests, then the host. A 12-drive pool deserves an orderly stop.

---

## Post-move sequence

Run these **in order**. Each step's failure mode masquerades as the next step's
problem if you skip ahead.

### 1. Storage before anything else

Standing guardrail after any host power event, and it applies doubly after a move —
a drive cable unseated in transit is the most likely casualty:

```sh
zpool status holocron    # ONLINE, both RAIDZ2 vdevs, 12 members, no DEGRADED/UNAVAIL
zpool list               # pool imported at all?
```

Do this before touching the network. A hard NFS mount on a missing pool produces
hangs elsewhere in the lab that look nothing like a storage fault.

### 2. Physical and link layer

- executor `enp0s31f6` → a plain **LAN** port on senate. Not the WAN/internet port,
  and not a multi-gig port configured as WAN.
- executor `enp5s0` → death-star port 1.
- `ip -br link` — confirm both names survived. NIC names encode PCI topology and this
  repo has two prior incidents of them moving.

### 3. Known-good comparison — is the house online at all?

Load a page from a phone on the new gateway's wifi. If that fails, stop: nothing in
the lab is broken and you are about to debug the wrong box. This single test halves
the search space and costs thirty seconds.

### 4. Establish the new gateway's LAN parameters

From that same device: subnet, mask, gateway, and — critically — **the DHCP pool
boundaries**. Then check whether anything already holds the lab's static addresses.

If the new gateway's LAN falls inside `10.0.0.0/16`, stop and renumber *its* LAN
before proceeding. The lab supernet cannot contain the house network: executor's
`10.0.0.0/16 via 10.0.10.1` route would swallow house traffic and tarkin would carry
overlapping connected routes.

### 5. Carve a static block the gateway cannot assign

Narrow the gateway's DHCP pool so the lab's WAN addresses sit outside it (ADR 0022).
Existing leases inside the newly-excluded range persist until renewal; they migrate
down on their own and nothing breaks meanwhile.

### 6. Rebuild senate-side state

- Static route `10.0.0.0/16` → tarkin's WAN address
- Reservations for tarkin's WAN and executor (belt and braces; being outside the pool
  is the real protection)
- Port forwards, if any

### 7. Clear the wreckage

If an address conflict occurred, flush what it left behind on tarkin:

```sh
pfctl -F states    # drop states built against a conflicting/old source
arp -d -a          # drop cached ARP entries, including any poisoned gateway mapping
```

Both self-clear eventually — states expire, ARP ages out in ~20 minutes — but after a
conflict you want a clean slate rather than a stale entry to debug. **Note:** this
drops every connection transiting tarkin, including browser sessions and WebSockets to
lab services. Expect UIs to need a hard reload. See the 2026-08-26 log.

---

## Verification ladder

Judge by reply codes and routing, not by "it works."

```sh
ping -c3 <senate gw>                 # layer 3 to the new gateway
ping -c3 8.8.8.8                     # routing, DNS not involved
dig @10.0.30.53 example.com          # NOERROR + A record — full recursion restored
dig @10.0.30.53 flurry.com           # 0.0.0.0 — gravity intact
dig @10.0.30.1  example.com          # NOERROR — isolates Unbound from Pi-hole
```

Then the parts that only fail at the edges:

- From a **house** device: `https://10.0.30.2:8006` — proves the static route *and* the
  WAN pass rule together. Verify both edges: the allow works and the wall still stands.
- `tailscale status` on phantom, then confirm echo-base's next replication pull lands.
  Offsite replication has been failing for the entire duration of the outage.
- Proxmox: confirm every VM has `onboot` set, or the next site move surprises you.

---

## Log

### 2026-08-26 — Relocation: no outbound connectivity at the new site

#### Issue
Lab came up at the new site with all internal services healthy and internal DNS
answering, but no outbound internet from any VLAN.

#### Root cause
Address conflict on `192.168.1.100`. The new ISP-supplied gateway serves the same
`192.168.1.0/24` LAN as the previous router, so every address in the repo remained
valid — but its DHCP pool spans `.2–.254` and its Option 60 vendor-class table had
already assigned `.100` to an ISP-supplied device that self-provisions. tarkin's
static `.100` was protected at the old site only by a reservation on the old router,
which did not travel. Two devices, one address, one L2 segment.

#### Fix
Gateway DHCP pool narrowed to `.2–.99`; tarkin WAN moved to `192.168.1.200`;
senate-side static route and port forward re-pointed. executor unchanged at `.225`.
Recorded as ADR 0022.

#### Learning
**A static IP is only safe if the DHCP authority on that segment agrees it is
off-limits.** On equipment you do not control, choose an address the server
structurally cannot assign rather than one you have asked it not to.

Second learning, on reading symptoms: internal DNS answering told us nothing, because
cache hits, gravity blocks and Kea-registered local names are all answered without
leaving the lab. Only recursion touches the internet. `dig` for a blocked name
returning `0.0.0.0` *and* an uncached public name returning SERVFAIL is the precise
signature of "resolver healthy, route dead" — and it was visible from the first
minute.

### 2026-08-26 — Two wrong predictions, recorded

**Prediction 1:** that the new site's LAN would be a different subnet, stranding
tarkin's static WAN on a network with no gateway — with a five-file re-address as the
fix. Falsified: the new gateway serves `192.168.1.0/24` as well.

**Prediction 2, after that correction:** that no lab-side change would be needed at
all, since every repo value stayed literally valid. Also wrong — right about the
subnet, wrong about the host address, because the ISP gateway claims `.100` through a
vendor-class mechanism the previous consumer router did not have.

Kept here because the second prediction was the more instructive failure: matching
syntax is not the same as an available address, and "nothing needs to change" is a
conclusion that deserves the same evidence as any other.

### 2026-08-26 — Immich UI unreachable (self-inflicted, not a fault)

#### Issue
vault's web UI at `http://10.0.30.40:2283` went unreachable during the network work.

#### Root cause
`pfctl -F states` on tarkin, run to clear the address conflict. falcon (VLAN 20) reaches
vault (VLAN 30) *through* tarkin, and Immich's UI holds a WebSocket open; flushing the
state table dropped it. The page stopped updating rather than cleanly erroring, which
reads as a dead service.

#### Evidence it was not a storage or service fault
- `docker compose ps` — all four containers up
- `mnt-photos.mount` ActiveEnterTimestamp **earlier** than `immich_server` StartedAt —
  storage was ready before the container started, so no boot-ordering race
- VM 107 up throughout, `onboot` enabled
- `zpool status holocron` clean, 12 disks online

#### Learning
Flushing the state table on the router is not a read-only diagnostic — it is a
disconnection event for every service behind it. Worth saying out loud before running
it, and worth reaching for a hard reload before opening a service investigation.

The ADR 0015 hard-mount design was never exercised here, so its falsifier (a vault NFS
stall affecting anything outside vault) remains untested rather than confirmed.

---

## Deferred / follow-ups

- **Offsite replication not yet verified post-move.** echo-base stayed at its original
  site while the primary site moved, so the correlated-regional-risk passage in
  `docs/threat-model.md` has been rewritten: separation improved, physical access
  degraded. Two things remain unconfirmed — that the first post-move pull actually
  lands, and how its throughput compares to the 2026-08-14 seed (~76 MiB/s). The path
  now crosses a different ISP at the primary end; a slower pull lengthens the effective
  RPO. Replication was failing for the entire duration of the outbound outage.
- **Bridge/passthrough mode on senate** — rejected in ADR 0022 for now. Revisit if the
  ISP gateway proves unmanageable.
- **Port forward viability** — confirm the gateway's WAN address is not CGNAT
  (`100.64.0.0/10`). If it is, the 25565 forward cannot work at this site regardless of
  configuration, and that is a decision to record rather than a fault to chase.
