# Phase 3 — DNS Filtering & Tailscale Remote Access

**Goal:** Put every VLAN behind filtered, private DNS — order66 (Pi-hole) resolving through
recursive Unbound on tarkin — with per-client query visibility, and add zero-exposure
remote access through a single Tailscale gateway (phantom). Along the way: a lease-table
audit that made the documented reservations real and caught a trust-zone violation, and
the offsite replication mechanism decided (ADR 0010).

**Status:** ✅ Complete — the deferred offsite appliance was built in Phase 5 (2026-08-14)

---

## Decisions & Rationale

### Why "Listen on all interfaces, permit all origins" on Pi-hole?

Pi-hole's default local-only listening mode refuses queries arriving from other subnets —
useless for a resolver serving five VLANs from VLAN 30. The permissive mode is deliberate
and is not an exposure: WHO may reach port 53 is enforced at tarkin's firewall, the correct
layer for a reachability decision, and order66 has zero inbound exposure beyond what those
rules pass. The resolver answers whoever the firewall admits; the firewall decides who that is.

### Why recursion took more than flipping Unbound's mode

Clearing Query Forwarding makes Unbound recursive, but two other paths can silently
re-inject a forwarder: the System → Settings → General DNS list, and "Allow DNS server list
to be overridden by DHCP/PPP on WAN." tarkin's WAN takes a DHCP lease from senate, so a
lease renewal could quietly put a forwarder back. Emptied the list, unchecked the override,
left DNS-over-TLS empty, enabled DNSSEC. Recursion now can't silently degrade back to
forwarding.

### Why SECLAB was left out of the cutover

The phase-scope note said "all 6 subnets," but network/dns-design.md scopes the cutover to
five and defers SECLAB to Phase 7 — the repo is the source of truth, so the repo won.
Reality-check while there: SECLAB's ruleset matches network/firewall-rules.md
(block → 10.0.0.0/16 on top, then pass any), which means no port-53 exception exists and
the DHCP-provided DNS (10.0.60.1) is unreachable — the VLAN has no working DNS by default.
Left deliberately: nothing lives there yet, and Phase 7 should design SECLAB's DNS posture
on purpose rather than inherit one.

### Why Kea reservations for naming — and why the plugin was declined

OPNsense's Kea does not register dynamic lease hostnames in Unbound; only static
reservations register, and only on an Unbound restart. The third-party os-kea-unbound
plugin closes that gap, but it is an unsigned, manually installed package running on the
firewall — an unacceptable supply-chain trade for a cosmetic gain. Strategy instead: Kea
reservations for devices worth naming; everything else stays an IP in the logs.

### Why one Tailscale gateway instead of an agent on executor (ADR 0011)

phantom is the only tailnet ingress: subnet router for 10.0.30.0/24 plus exit node, with no
tailscaled anywhere else. An agent on executor was rejected — an agent on the
highest-value host means its compromise is total. The MGMT VLAN is not advertised at all;
the hypervisor is reached over the servers route at https://10.0.30.2:8006 (the ADR 0009
storage leg — pveproxy listens on all host interfaces). An unadvertised network beats an
advertised-but-ACL'd one: structural control over policy control. The pveproxy exposure
this relies on is a documented accepted risk (docs/threat-model.md).

### Why tailnet ACLs stay default allow-all (for now)

Single-user tailnet — every device on it is mine. The operative controls today are device
authorization, key management (expiry disabled so a silent key expiry can't cut off the
headless gateway; devices leave by explicit revocation), and MGMT staying structurally
unadvertised. ACLs would add policy to a trust boundary that currently has one principal.
Revisit triggers, recorded in ADR 0011: any non-owner device joins, or Phase 6 hardening.

### Why the tailnet resolves through Pi-hole (DNS override)

Admin-console global nameservers 10.0.30.53 primary and 10.0.30.1 secondary (secondary
added 2026-07-14 to mirror the LAN posture) with "Override DNS servers" enabled: every
tailnet device except phantom resolves through Pi-hole via the subnet route whenever
Tailscale is up — filtering follows remote devices out of the house. phantom itself runs
`--accept-dns=false`: the gateway carrying everyone's DNS must never accept a DNS push
itself, or it resolves through the tunnel it provides (loop prevention). Known trades:
per-client attribution collapses to phantom (the subnet router SNATs), and remote DNS
depends on lab availability — escape hatch is toggling Tailscale off.

### Why offsite replication is pull-based syncoid (ADR 0010)

The offsite copy exists to survive ransomware, so the lab must not hold a credential that
can reach it. The Pi 5 pulls via syncoid (ZFS send/recv orchestration) over Tailscale and
holds the only credential — a delegated zfs-allow user on archives — so a compromised lab
cannot reach, let alone destroy, the offsite history. Push designs were rejected exactly
because a lab-side credential can; cloud stays rejected on cost (ADR 0005). Fallback if
ZFS-on-USB misbehaves: restic to an append-only rest-server. Implementation deferred
pending hardware (drive enclosure).

### Why phones have no reservations

Modern phones randomize per-SSID (locally administered) MACs. A reservation pinned to a
rotating MAC breaks silently the day the MAC rotates — worse than no reservation. Phones
stay dynamic; reservations are for stable-MAC devices only.

---

## Checklist

| Step | Status | Notes |
|------|--------|-------|
| order66 VM created (ID 104) | ✅ | Ubuntu Server 24.04, VLAN 30, 10.0.30.53 |
| Pi-hole v6 installed + configured | ✅ | Upstream 10.0.30.1; DHCP off (Kea owns 67); StevenBlack list |
| Conditional forwarding + local domain | ✅ | 10.0.0.0/16 → 10.0.30.1; galaxy.internal |
| Unbound recursive + DNSSEC | ✅ | dnssec-failed.org → SERVFAIL |
| WAN forwarder re-injection closed | ✅ | General DNS list emptied; WAN DHCP override unchecked |
| FAMILY/IOTGUEST port-53 exceptions | ✅ | → .53 and → .1 above the /16 block; old → 10.0.x.1 deleted |
| Kea DNS cutover (5 subnets) | ✅ | TRUSTED first as canary; SECLAB deferred by design |
| Per-client attribution verified | ✅ | Pi-hole query log, across VLANs |
| Zero-upstream incident fixed + verified | ✅ | Headline log entry below |
| Genuine failover drill | ✅ | Re-run 2026-07-14 post-fix — PASSED |
| Lease-table audit + reservations | ✅ | shipyard, falcon, 4× IoT, fire-tv; scout hostname fixed |
| phantom rebuilt (ID 101) | ✅ | Subnet router 10.0.30.0/24 + exit node |
| Tailnet DNS override → Pi-hole | ✅ | Verified on-net (laptop) and off-net (phone, LTE) |
| ADRs 0010 + 0011 recorded | ✅ | Offsite mechanism; remote-access gateway pattern |
| Pi 5 offsite appliance | ✅ | Built in Phase 5 as `echo-base`, operational 2026-08-14 (ADR 0010) |

---

## order66 VM Spec

| Setting | Value |
|---------|-------|
| VM ID | 104 |
| Hostname | order66 |
| OS | Ubuntu Server 24.04 LTS |
| Disk | 16GB, NVMe store |
| CPU | 2 vCPU |
| RAM | 2048MB |
| Network | vmbr1, VLAN tag 30 |
| IP | 10.0.30.53 (in-OS static) |

Pi-hole v6 — upstream 10.0.30.1 (Unbound on tarkin); Pi-hole DHCP disabled (Kea owns
port 67 — guardrail); default StevenBlack blocklist; "Listen on all interfaces, permit all
origins"; conditional forwarding 10.0.0.0/16 → 10.0.30.1, local domain `galaxy.internal`
(`.internal` is the ICANN-reserved private-use TLD).

## phantom VM Spec

| Setting | Value |
|---------|-------|
| VM ID | 101 |
| Hostname | phantom |
| OS | Ubuntu Server 24.04 LTS |
| Disk | 12GB, NVMe store |
| CPU | 2 vCPU |
| RAM | 1024MB |
| Network | vmbr1, VLAN tag 30 |
| IP | 10.0.30.10 (in-OS static) |

Tailscale gateway — IP forwarding via `/etc/sysctl.d/99-tailscale.conf`; `tailscale up`
with `--accept-dns=false`; advertised routes = 10.0.30.0/24 only, plus exit node; key
expiry disabled on all tailnet devices; auto-enrolled in the nightly nas-vmbackups vzdump
job via its "all except" selection (only 100 and 102 excluded, per ADR 0009).

---

## Log

### 2026-07-13 — order66 build + network-wide DNS cutover

Created order66 (VM 104) per the spec above and installed Pi-hole v6: upstream 10.0.30.1
(Unbound on tarkin), Pi-hole's DHCP left disabled (Kea owns port 67 — guardrail), default
StevenBlack blocklist. Set listening to "Listen on all interfaces, permit all origins" and
added conditional forwarding 10.0.0.0/16 → 10.0.30.1 with local domain `galaxy.internal`,
so lab reverse lookups and bare hostnames resolve via Unbound.

Switched Unbound on tarkin from forwarding to full recursion: Query Forwarding cleared,
"Use System Nameservers" unchecked, DNS-over-TLS list empty, DNSSEC enabled. Closed the
re-injection paths: System → Settings → General DNS list emptied and "Allow DNS server
list to be overridden by DHCP/PPP on WAN" unchecked (senate could otherwise re-inject a
forwarder on a WAN lease renewal). Validation verified: dnssec-failed.org → SERVFAIL.

Firewall and DHCP cutover: on FAMILY and IOTGUEST, added pass TCP/UDP → 10.0.30.53:53 and
→ 10.0.30.1:53 above the block → 10.0.0.0/16. Kea set to primary 10.0.30.53 / secondary
10.0.30.1 on five subnets (10/20/30/40/50), routers unchanged, auto-collect OFF with
values set manually (guardrail). TRUSTED cut over first as canary, then the rest.
Per-client attribution verified across VLANs in the Pi-hole query log. The old
→ 10.0.40.1:53 and → 10.0.50.1:53 exceptions deleted after cutover verification. SECLAB
(60) intentionally untouched — see Decisions & Rationale.

### 2026-07-13 — Headline incident: silent zero-upstream failure

#### Issue
Pi-hole had ZERO upstream DNS servers enabled — since install. Every non-blocked query
returned SERVFAIL. The network was fully functional the entire time, so nothing surfaced it.

#### Root cause
The upstream selected at install never persisted (underlying cause unknown). The masking
mechanism is the interesting part: Kea hands out secondary 10.0.30.1, so every capable
client silently failed over to Unbound directly — while blocked domains STILL returned
0.0.0.0, because FTL answers blocks locally with no upstream needed. Spot-checks of
blocking therefore "passed." The designed path was dead; the user experience was perfect.

#### Fix
Settings → DNS → Custom upstream 10.0.30.1, applied 2026-07-13. Verified from a client:
`dig @10.0.30.53 example.com` → NOERROR via Unbound recursion (EDE 3 "Stale Answer" noted —
benign serve-stale behavior; EDE = Extended DNS Errors, RFC 8914), and
`dig @10.0.30.53 flurry.com` → 0.0.0.0 with EDE 15 (Blocked — gravity).

#### Learning
"It works" is not "it works via the path you designed." A well-built fallback masks primary
failure perfectly; verify the reply-code distribution, not the user experience. SIEM
candidate rule (Phase 6): SERVFAIL-ratio alert on Pi-hole logs.

**Consequence:** the 2026-07-13 fallback drill was retroactively vacuous — clients were
already living on the secondary the whole time. Re-run post-fix on 2026-07-14 (below).

### 2026-07-13 → 2026-07-14 — Incident: Alexa lost the Magic Home LED strip (closed)

#### Issue
Alexa control of the Magic Home LED strip stopped after the DNS cutover. Both devices
online; local app control on VLAN 50 unaffected — only the Alexa→strip leg broke.

#### Root cause
A downstream symptom of the zero-upstream incident, exposed by asymmetric client behavior.
The strip (ESP-based, 10.0.50.100 at the time) queried primary .53, received SERVFAIL,
and — unlike the Echo — never failed over to the secondary. Its session to the vendor
cloud died and stayed dead. The Echo's resolver failed over instantly, and the phone app
uses local LAN control, so every other leg kept working.

#### Fix
The upstream fix plus power-cycling the strip. Full Alexa↔strip functionality returned
immediately and remains stable after reservation adoption (led-strip, 10.0.50.11).

#### Learning
Cheap IoT DNS clients often pin the first DHCP-provided resolver and never fail over —
asymmetric client behavior turned one server-side fault into a confusing
single-integration break. Wrong turns worth recording: allowlisting generic Amazon domains
(the Echo's leg was never broken), and searching the query log for BLOCKS when the failure
mode was SERVFAIL replies on allowed queries.

### 2026-07-13 — phantom rebuild: Tailscale gateway (subnet router + exit node)

Destroyed and recreated VM 101 per the spec above. Enabled IP forwarding via
`/etc/sysctl.d/99-tailscale.conf`; brought Tailscale up with `--accept-dns=false` (the
gateway must never take tailnet DNS pushes — loop prevention). Subnet route and exit node
approved in the admin console; key expiry disabled on ALL tailnet devices. phantom joined
the nightly nas-vmbackups vzdump job automatically via its "all except" selection.

#### Issue: no remote Proxmox access after the first route advertisement
**Root cause:** only 10.0.30.0/24 was advertised, which carries no path to the Proxmox UI
on MGMT.
**Fix:** fixed LIVE from the remote side — Tailscale SSH into phantom, re-advertised with
10.0.10.0/24 added, approved the new route in the admin console.
**Learning:** `--advertise-routes` REPLACES the list rather than appending — re-state
every route each time — and each newly advertised route carries nothing until it gets
fresh admin-console approval.

**Deliberate reversal (same day):** removed 10.0.10.0/24 again. An unadvertised network
beats an advertised-but-ACL'd one — structural control over policy control. Final state:
routes = 10.0.30.0/24 only, plus exit node. Remote hypervisor management moved to
https://10.0.30.2:8006 — pveproxy listens on all host interfaces, so the UI is reachable
on the ADR 0009 storage leg over the servers route. That is now the designated remote
management path and a documented accepted exposure (docs/threat-model.md).

Tailnet DNS override: admin-console global nameservers 10.0.30.53 primary and 10.0.30.1
secondary (secondary added 2026-07-14 to mirror the LAN posture) with "Override DNS
servers" enabled — every tailnet device except phantom resolves through Pi-hole via the
subnet route whenever Tailscale is up. Verified on the laptop via `tailscale dns status`
(both resolvers listed, node accepting) and off-network on the phone (LTE, Tailscale on):
queries appear in the Pi-hole log attributed to phantom; blocked domains fail.

#### Issue: with the override enabled, a remote nslookup still returned public IPs for a blocked domain
**Root cause (probable):** the override/nameserver was not yet in effect at test time —
not conclusively pinned. `tailscale dns status` later confirmed the node accepts tailnet
DNS with resolver 10.0.30.53. The test was also confounded by the then-undiscovered
zero-upstream incident: even a correctly routed query would have SERVFAILed.
**Fix:** none required — the verification above passed once the override was in effect
and the upstream was fixed.
**Learning:** on systemd-resolved systems, /etc/resolv.conf always shows the 127.0.0.53
stub — it proves nothing. Read `resolvectl status` / `tailscale dns status` for the real
uplinks, and judge DNS paths by the ANSWERS returned, not the server field.

### 2026-07-13 → 2026-07-14 — Lease-table audit: reservations, identities, zone violations

Audited the Kea lease table against the intended statics and reservations in
network/ip-scheme.md. Findings below; reservations created 2026-07-14, with Unbound
restarted afterward so the names register.

#### Issue: shipyard held two addresses — its static .25 plus a dynamic lease at 10.0.30.100
**Root cause:** confirmed 2026-07-14 — netplan carried `dhcp4: true` alongside the static
config, so the NIC held both addresses (same MAC, `bc:24:11:43:1d:52`, in the lease
table). The NAT rule and services were always at 10.0.30.25; external Minecraft never
broke, and ip-scheme.md was accurate.
**Fix:** removed `dhcp4`, re-applied netplan; only .25 remains. Kea reservation for
10.0.30.25 created as pool-collision protection and for Unbound name registration.
**Learning:** an interface can hold a static and a DHCP lease simultaneously; audit the
lease table against intended statics periodically.

#### Issue: falcon's documented reservation at 10.0.20.10 never existed
**Root cause:** ip-scheme.md claimed a reservation that was never created — falcon was
dynamic at 10.0.20.105, self-reporting "cole10".
**Fix:** reservation created 2026-07-14 — initially mis-created with the Mint laptop's MAC
(Intel `f4:6d:3f:xx:xx:xx`, self-reporting "evil"), caught by MAC-vendor cross-reference
and corrected to falcon's ASRock NIC `9c:6b:00:xx:xx:xx`. Adopted via release/renew;
falcon now holds 10.0.20.10.
**Learning:** identify devices by self-reported hostname + MAC vendor before pinning; the
lease table is evidence, not decoration.

#### IoT reservations
Created 2026-07-14: echo 10.0.50.10 (`b4:7c:9c:xx:xx:xx`), led-strip 10.0.50.11
(`c4:4f:33:xx:xx:xx`), levoit-purifier 10.0.50.12, levoit-humidifier 10.0.50.13. Echo and
strip power-cycled and confirmed adopted at their reserved addresses 2026-07-14.

#### Issue: unknown Amazon device on TRUSTED — a Fire TV stick in the wrong trust zone
**Root cause:** the Fire TV stick (10.0.20.100, `80:6d:71:xx:xx:xx`) had joined a TRUSTED
SSID — Amazon hardware belongs on IOTGUEST per the trust-zone policy.
**Fix:** moved to MosEisley and reserved at 10.0.50.14 (fire-tv); adoption confirmed
2026-07-14. Caveat: the Fire TV phone-remote app discovers over the local network, so it
requires the phone on MosEisley; Echo voice control is cloud-side and unaffected.
**Learning:** SSID sprawl quietly defeats VLAN policy — periodic lease audits catch zone
violations.

#### Issue: laptop identity split — OS hostname "evil" vs repo name "scout"
**Root cause:** the OS hostname of the Mint laptop never matched the inventory name.
**Fix:** OS hostname changed to scout 2026-07-14 (hostnamectl + /etc/hosts); tailnet
machine name updated to scout. scout remains DHCP on VLAN 20 per plan (Phase 7 assigns
10.0.60.10, ADR 0007). The audit also observed historical leases for this MAC on both
TRUSTED and FAMILY — the laptop had previously joined OuterRim; harmless, noted.
**Learning:** device OS hostnames must match the inventory, or log attribution degrades.

Phones with randomized (locally administered) MACs were deliberately NOT reserved — a
reservation pinned to a rotating MAC breaks silently.

### 2026-07-14 — Genuine failover drill (re-run post-fix): PASSED

With the upstream fixed, re-ran the drill the zero-upstream incident had invalidated:
order66 cleanly shut down → clients continued resolving via 10.0.30.1 → order66
restarted → primary path resumed. This was the first genuine test of the designed
failover — the 2026-07-13 run exercised nothing, because clients were already living on
the secondary.

---

## Open items

- ~~**Pi 5 offsite appliance**~~ — **closed 2026-08-14.** Built as `echo-base` during
  Phase 5 and replicating `holocron/photos` + `holocron/media`; restore test passed. See the
  [ADR 0010 implementation amendment](../decisions/0010-offsite-replication-mechanism.md),
  the [Phase 5 log](phase-5-vault-immich.md), and
  [runbooks/offsite-replication.md](../runbooks/offsite-replication.md). Two prerequisites
  named here resolved differently than planned: `holocron/media-keep` was superseded (media
  is small enough to replicate whole) and `holocron/configs` was never created, so configs
  still have no offsite copy — that gap is now tracked in ADR 0010.
- **SECLAB DNS** — none by design; posture defined in Phase 7.
- **pveproxy exposure mitigations** — pve-firewall or a LISTEN_IP bind; from-home tasks,
  deferred (accepted risk in docs/threat-model.md).
- **SIEM rule seed (Phase 6)** — SERVFAIL-ratio alert on Pi-hole reply codes, from the
  headline incident.
