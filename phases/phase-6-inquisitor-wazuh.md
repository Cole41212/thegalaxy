# Phase 6 — inquisitor (Wazuh SIEM) & the Notification Path

**Goal:** Build inquisitor (VM 105, 10.0.30.60) as the lab's Wazuh SIEM — manager, indexer,
and dashboard — with agents on every host whose OS the lab controls and syslog from the two
appliances it does not. And *first*, before any of that: restore a working, **tested**
notification path, which the lab did not have. Along the way: the deployment model and sizing
(ADR 0018), data placement and what an SIEM backup is actually for (ADR 0019), agent coverage
and the log-source split (ADR 0020), and the alerting design itself (ADR 0021).

**Status:** 🚧 In progress — Step 0 (the notification path) and the pre-build hygiene work are
done, inquisitor is built, and Wazuh 4.14.7 is running all-in-one. Remaining: the alert
transports beyond mail (Pushover, the deadman, the PVE matchers), the agent rollout, and
detection content. Two items are open and unresolved: a memory-PSI regression on executor that
got *worse* after the overcommit fix, and **ISP interception of all outbound port 53**
(2026-09-26/27) which killed recursion and forces the resolver onto DNS-over-TLS — design
verified, ADR unwritten, with a hard 2026-10-11 deadline on restoring DNSSEC validation.

---

## Decisions & Rationale

Four ADRs carry this phase. [ADR 0018](../decisions/0018-wazuh-deployment-model-and-sizing.md)
settles the deployment model and sizing — all-in-one on one VM, and why the host's real memory
constraint is worse than the allocation table suggests.
[ADR 0019](../decisions/0019-siem-data-placement-and-backup.md) settles where index data lives
and what is worth backing up. [ADR 0020](../decisions/0020-agent-coverage-and-log-sources.md)
settles which hosts get an agent, which get syslog, and what that makes inquisitor in
blast-radius terms. [ADR 0021](../decisions/0021-alerting-and-notification-path.md) is the
phase's central decision and gets the most space there.

### Alerting before detection

The build order in this phase is deliberately inverted from the obvious one. Wazuh is the
headline; the notification fix was done first.

The reason is that a detection engine bolted to an untested channel produces **unverifiable
rules**. A rule that fires into a black hole and a rule that never fires are indistinguishable
from outside the system, so every rule written before the channel is proven has to be re-proven
afterwards anyway — and in the meantime the SIEM produces a false sense of coverage, which is
worse than no coverage, because it stops you looking.

This is not a hypothetical. On 2026-08-02 a drive in holocron faulted during the weekly scrub,
Proxmox generated the alert correctly, and nobody received it; the fault was found by hand days
later (Phase 5 log). Every layer of detection worked. The delivery path was broken and had been
broken for months. [docs/telemetry-logging.md](../docs/telemetry-logging.md) already carried the
line this phase is built on — **"an alert channel nobody has tested is not a control"** — and
this is the phase that acts on it rather than restating it.

So: mail first, push second, deadman third, and each proven by a deliberately triggered real
event before a single decoder was written. The full design, including why no alert path may
share a failure domain with what it monitors, is in
[ADR 0021](../decisions/0021-alerting-and-notification-path.md).

### Phase 6 begins with no back-history

[docs/telemetry-logging.md](../docs/telemetry-logging.md) planned an interim syslog sink — a
lightweight collector, or inquisitor early in syslog-only mode — to be stood up when Phase 3
landed order66, so that firewall, DNS, and auth history would accumulate before detection rules
had to be written against it. **That sink was never built.** Phases 3, 4, and 5 all shipped
without a central collector, so none of that history exists.

This is recorded as an **accepted condition**, not as a task to catch up on. There is no way to
retroactively generate three phases of telemetry, and waiting to accumulate a baseline before
writing any rules would defer the entire phase for months while continuing to run with no
detection at all.

The consequence is a change of method rather than a change of scope. Rules get tuned against
**deliberately triggered events** instead of against a historical baseline: generate the thing
the rule is supposed to catch — a failed SSH login, a blocked inter-VLAN attempt, a file change
under a watched path — capture the **real log line** it produced, and test the decoder and rule
against that captured line with `wazuh-logtest`. That tool takes a raw log line on stdin and
shows the decoder that claimed it, the fields extracted, and the rule that matched, which makes
rule development a closed loop that does not need history.

What is genuinely lost is the ability to say "this is unusual for this host" on day one.
Frequency-based and baseline-derived rules have to wait for the lab to generate their own
history, which starts accruing the moment agents connect. Signature-shaped rules do not wait.

### executor resolves DNS outside the virtualised stack

executor's resolver stays **192.168.1.1 (senate)** rather than order66, and this is a decision
rather than an oversight.

postfix on executor needs DNS to resolve `smtp.gmail.com` before it can deliver anything.
Pointing the hypervisor's resolver at a guest VM would mean that **order66 going down could
suppress the alert saying order66 is down** — the alert path would run through the thing it is
reporting on. That is the same structural rule as ADR 0009's host storage leg (host↔NAS traffic
must not transit the router VM it might need to report about) and the same rule as keeping the
deadman on an external service. A monitoring path that shares a failure domain with its subject
is not a monitoring path.

**A single nameserver, deliberately.** No secondary is configured. Phase 3's headline incident
was precisely a fallback masking a dead primary: Pi-hole had zero working upstreams and nothing
surfaced it, because every client silently used the DHCP-provided secondary and the network
stayed fully functional the entire time. A resolver failure on executor should be *loud*, and a
fallback is what makes it quiet.

Two consequences follow, both accepted:

- **executor cannot resolve `.galaxy.internal`.** senate knows nothing about the lab's private
  zone. So executor's Wazuh agent is configured with the **manager's IP address** rather than
  its FQDN — agent configuration takes an address, and hard-coding one on the host that cannot
  resolve names is correct rather than lazy.
- **executor's DNS queries do not appear in Pi-hole.** The per-client visibility that
  [network/dns-design.md](../network/dns-design.md) exists to provide has a hole in it exactly
  where the hypervisor is. Host DNS visibility therefore comes from **journald ingestion** via
  the Wazuh agent instead of from Pi-hole's query log — a different source answering the same
  question, recorded in the source table in
  [ADR 0020](../decisions/0020-agent-coverage-and-log-sources.md) so the gap is not rediscovered
  later as a surprise.

### ssd-inquisitor stays on the weaker drive

The SIEM's 200 GiB data disk sits on the Crucial BX200 — measurably the more worn and
structurally the weaker of the host's two SATA SSDs — and the Samsung 860 EVO keeps
`ssd-vmstore`. The SMART baseline taken 2026-08-17 (log below) made the case for swapping them
concrete enough to evaluate, and the answer was **no swap**, for three reasons:

1. **The workload does not bind.** Wazuh at this scale writes roughly 200 MB/day against ~68 TB
   of remaining endurance on the BX200. The drive's known weakness is *sustained* steady-state
   write throughput; a trickle of index writes never enters that regime.
2. **The heavier writer is already on the better drive.** The nightly vzdump job for tarkin and
   archives is a burst of many GiB against ssd-vmstore — exactly the sustained-write shape the
   BX200 is bad at. Moving the SIEM onto the Samsung would mean moving vzdump onto the BX200.
3. **The data on each drive has opposite value.**
   [ADR 0019](../decisions/0019-siem-data-placement-and-backup.md) declares index data
   expendable — alerts are derived evidence and the source logs remain on the endpoints —
   while [ADR 0009](../decisions/0009-infrastructure-vm-backups-local.md) makes ssd-vmstore the
   **only** local copy of the router and NAS configurations. The more valuable data belongs on
   the better drive, and it already is.

The finding that actually matters from that baseline is not the wear level but the
instrumentation: every SMART threshold on the BX200 is `000`, so its own self-assessment is
structurally incapable of ever reporting failure. **Monitoring must read raw attribute values,
not normalized ones** — the Phase 6 SMART rule for `ssd-inquisitor` is written accordingly, and
the replacement trigger is recorded in
[docs/hardware-inventory.md](../docs/hardware-inventory.md).

### Let the vendor installer install stock, then adapt

Both Wazuh installer failures were self-inflicted, and both trace to one mistake: configuration
staged for software that did not exist yet. The index disk was prepared, mounted, and wired
into systemd *before* the installer ran. The installer's own "is Wazuh already installed?"
check is a bare directory test, which the empty, correctly-prepared mountpoint satisfied with
no packages present anywhere. The second attempt then failed the opposite way — the mount
activated at service start and shadowed the directory the installer had just populated.

The rule taken from it: **let an opinionated vendor installer install stock, confirm it works,
then adapt.** Preparation ahead of such an installer looks like diligence and behaves like a
defect, because there is nothing yet to validate it against — so its errors surface as
installer failures two layers away from their cause.

The complementary discipline is the half that paid: reading the installer *before* running it.
That is how the 4.14.7 heap defect was found in the same sitting, rather than meeting it later
as an unexplained performance ceiling with no error attached.

### Secure Boot stays on — the passthrough runbook's rule is TrueNAS-specific

inquisitor boots with Secure Boot enabled and verified, which on its face contradicts
[runbooks/pcie-passthrough.md](../runbooks/pcie-passthrough.md): "uncheck Pre-Enroll keys so
Secure Boot is OFF." That instruction is correct **for TrueNAS**, which is unsigned and was
rejected by OVMF in Phase 2, and the runbook states that reason in its parenthetical. Ubuntu
Server 24.04 ships a signed shim and kernel, so the Phase 2 finding does not transfer, and
inquisitor has no passthrough devices at all — the runbook was simply not written about it.

Recorded as a deliberate deviation rather than left to read as an inconsistency, because
anyone reading the passthrough runbook and this build spec side by side will otherwise assume
one of them is wrong.

### Memory is a host-level budget, not a guest-level setting

Two findings collapse into one principle.

**The correction first.** An earlier working claim — that the indexer's `bootstrap.memory_lock`
protects its heap from being paged out — is wrong at the layer that matters. `memory_lock` pins
the heap inside the guest's own address space, which stops the *guest* kernel swapping it. To
executor, inquisitor is a single 8 GB QEMU process, and the host may page out any part of that
process regardless of what the guest has locked. Only two mechanisms reach the host layer: PCI
passthrough, which pins the entire guest (ADR 0018's context), or simply not overcommitting.

**Then the measurement.** executor was found allocating 61.0 GiB against 62.5 GiB usable, with
5.9 GiB actually swapped — and the most-swapped guest, at 30%, was **tarkin, the router**. That
is the predictable outcome rather than bad luck: idle infrastructure has the coldest pages, the
kernel evicts by coldness, and so the guests whose stalls do the most damage are precisely the
ones most likely to be evicted. Corrected to 53.0 GiB allocated with zero swap.

**New guardrail:** every new VM is checked against the host memory budget *before* it is
created, not after it misbehaves.

The honest footnote is that this has not yet produced the improvement it predicted — memory PSI
got worse the following night, not better. That is open and instrumented, below.

---

## Checklist

| Step | Status | Notes |
|------|--------|-------|
| ADRs 0018–0021 recorded | ✅ | Deployment/sizing · data placement/backup · agent coverage · alerting |
| Proxmox alert mail restored | ✅ | 2026-08-16 — postfix authenticated satellite relay; queue flushed, `250 2.0.0 OK … gsmtp` |
| PVE native SMTP notification target | ⬜ | Second, postfix-independent path off the same host (ADR 0021) |
| PVE notification matchers | ⬜ | Route vzdump **failures** to P2, **suppress successes** — before the first nightly run lands |
| TrueNAS native email configured + tested | ⬜ | archives uses its own mail path, not executor's relay |
| Pushover P1 path proven | ⬜ | Including emergency-priority repeat-until-acknowledged behaviour |
| Deadman checks live (executor + inquisitor) | ⬜ | Hourly ping, 90-minute grace; two checks, not one |
| Quarterly re-test scheduled | ⬜ | Triggered real events, not vendor test buttons |
| `pvescheduler` journal noise cleared | ✅ | 2026-08-16 — root-caused to an unclean shutdown; silent over 20 cycles |
| `feetwifi` hostname purge | ✅ | 2026-08-16 → 17 — hosts, mailname, search domain, aliases, **and the TLS certificate** |
| Storage pre-flight | ✅ | 2026-08-17 — both mounts held across device-letter drift; `is_mountpoint 1`; content types narrowed |
| SMART baseline, both SSDs | ✅ | 2026-08-17 — both PASSED; replacement trigger recorded for the BX200 |
| `holocron/configs` dataset + syncoid | ⬜ | ADR 0019 — also closes Phase 5 deferred item 6 |
| inquisitor built (VM 105) | ✅ | Secure Boot **ON** — deliberate deviation (above); 10.0.30.60 `dig`-verified before install |
| `vm.max_map_count=262144` persistent | ⬜ | Prerequisite, not tuning — the all-in-one installer normally sets it; **confirm it persisted** in `/etc/sysctl.d/` |
| Wazuh 4.14.x all-in-one installed | ✅ | **4.14.7** — manager + indexer + dashboard. Two installer failures first, both self-inflicted (log below) |
| 200 GiB data disk, **Backup unchecked** | ✅ | `backup=0`; raw — no partition table, reserved blocks reclaimed, mounted by UUID with `nofail` + `RequiresMountsFor` |
| Indexer heap `Xms=Xmx=3g` | ✅ | Set **manually** — 4.14.7 configures a 1 GB heap regardless of RAM (upstream defect, log below) |
| `number_of_replicas: 0`, archives indices off | ⬜ | Single node cannot allocate a replica; archives indices store every event |
| ISM policy — rollover + 90-day delete | ✅ | 90-day delete across 26 time-series indices; current-state indices permanently excluded |
| authd enrolment password set | ⬜ | Not open registration |
| `remote_commands` disabled · active response off | ⬜ | ADR 0020 blast-radius mitigations |
| Agents enrolled — 8 hosts | ⬜ | inquisitor, executor, shipyard, vault, cantina, order66, phantom, falcon |
| Syslog from tarkin + archives | ⬜ | TCP, restricted with `allowed-ips` |
| death-star syslog capability confirmed | ⬜ | Establish whether the switch can export at all before planning around it |
| Rules tuned with `wazuh-logtest` | ⬜ | Against captured real log lines from triggered events |
| SMART rule on `ssd-inquisitor` | ⬜ | **Raw** attribute values — normalized values on this drive can never fail |
| Immich dump freshness check | ⬜ | Configured **locally on vault** (`remote_commands` is off by design) — Phase 5 item 2 |
| SERVFAIL-ratio rule on order66 | ⬜ | Phase 3 seed |
| Dashboards on panel-1 / panel-2 | ⬜ | |
| Vulnerability detection enabled | ⬜ | **Found ON by default** — 13.7 GB on root before any agent existed; disabled and reclaimed. Re-enable after soak, headroom planned (log below) |
| Index sharding corrected | ✅ | Order-10 template — 3 shards/day → 1, verified twice; filebeat rewrites edits to its own |
| Unattended-restart hardening | ✅ | needrestart no longer auto-restarts Wazuh; `KillMode=control-group` drop-in; tested across two restarts |
| `net-tools` installed | ✅ | rootcheck's port check and rule 533's port-change monitoring were both broken without it |
| executor memory budget corrected | ✅ | 61.0 GiB / 5.9 GiB swapped → 53.0 GiB / zero swap; new pre-create gate |
| SIEM self-health check | ⬜ | `active` ≠ working — the wazuh-db socket outage ran 11 days undetected |
| Memory PSI regression | 🚧 | **Worse** after the fix — 22.2 s full stall vs ~7 s/day baseline. Instrumented, unresolved |
| DNS recursion — ISP interception | 🚧 | 2026-09-26/27 — all outbound :53 intercepted; recursion dead, forwarders unaffected. DoT on 853 verified; **ADR pending** before implementation |
| DNSSEC validation restored | ⬜ | **Deviation live** — hard deadline **2026-10-11** root KSK rollover |
| Pi-hole upstream returned to Unbound | ⬜ | **Deviation live** — tracked in network/dns-design.md; closes with the DoT ADR |

---

## inquisitor (VM 105) build config

As specified in [ADR 0018](../decisions/0018-wazuh-deployment-model-and-sizing.md), and as built —
deviations from the ADR are marked and reasoned through under Decisions & Rationale above.

| Setting | Value |
|---------|-------|
| VM ID | 105 |
| Hostname | inquisitor |
| OS | Ubuntu Server 24.04 LTS — **standard** install (cantina's Phase 4 lesson) |
| Machine | q35 |
| BIOS | OVMF (UEFI) + EFI disk — **Secure Boot ON**, verified (deliberate deviation) |
| CPU | 3 vCPU, host type; CPU units 50 |
| RAM | 8192 MB, ballooning **OFF** |
| System disk | 40 GB `scsi0` on `local-lvm` (NVMe) — discard + SSD emulation + iothread |
| Data disk | 200 GB `scsi1` on `ssd-inquisitor` — **Backup unchecked** (`backup=0`). Raw: no partition table, reserved blocks reclaimed, mounted by UUID with `nofail` + a `RequiresMountsFor` drop-in |
| Network | vmbr1, VLAN tag 30, VirtIO |
| Guest agent | qemu-guest-agent enabled |
| IP | 10.0.30.60 — DHCP in-OS + Kea reservation (cantina/vault pattern) |
| Kernel | `vm.max_map_count=262144`, set persistently |
| APT | `Acquire::ForceIPv4 "true"` at build time — third IPv6-first incident in this lab |
| Wazuh | 4.14.7 all-in-one — manager + indexer + dashboard |
| Indexer heap | `Xms=Xmx=3g`, set **manually** — the 4.14.7 installer computes 1 GB regardless of host RAM |

---

## Verification evidence

Evidence for the work completed so far. Each item is the *receiving* side of the transaction
where one exists, per ADR 0021's first principle.

### Mail path (2026-08-16)

- **Cause isolated in one test:** a paired reachability check against `smtp.gmail.com` — **587
  open, 25 blocked** — which separated "postfix is misconfigured" from "the network won't carry
  it" without touching a config file.
- **Delivery confirmed by the server, not the client:** the postfix log records `status=sent`
  with the remote server's own acceptance string, `250 2.0.0 OK … gsmtp`. The sending command's
  exit status is not evidence and was not treated as such.
- **Queue drained:** 20 deferred messages / 529 KB, spanning at least 2026-08-11 to 08-14, all
  nightly vzdump reports, flushed after the reconfiguration.
- **TLS is fail-closed:** `smtp_tls_security_level=encrypt`, not `may` — the connection fails
  rather than falling back to cleartext.

### Journal hygiene (2026-08-16)

- **The file was inspected as bytes, not as text:** `xxd` on
  `/var/lib/pve-manager/pve-replication-state.json` returned `0000` — two NUL bytes. `cat`
  showed nothing and would have supported the wrong diagnosis.
- **Timestamps carried the diagnosis:** preserved mtime 2026-06-06 20:12:01, against boot
  session -14 beginning 21:01:25 — the write landed 49 minutes *before* the boot that followed
  it.
- **Fix verified by absence over time:** silent across 20 consecutive 60-second scheduler
  cycles.
- **No data was at risk:** `/etc/pve/replication.cfg` is absent — no replication jobs are
  configured — so resetting the state file to `{}` was lossless.

### `feetwifi` purge (2026-08-16 → 17)

- **The certificate is the item that mattered:** `pvecm updatecerts --force` regenerated the
  node certificate *after* `/etc/hosts` was corrected, so the new CN derives from
  `pve.galaxy.internal`. Ordering was load-bearing — the certificate takes its CN by resolving
  the node name.
- **Search discipline corrected:** the diagnostic now used is
  `grep -rq PATTERN /etc/ && echo FOUND || echo CLEAN`, which prints on both branches. The
  earlier form could print nothing at all (log entry below).
- **Residual, accepted:** the regenerated certificate's SAN covers **192.168.1.225 only** — not
  10.0.10.10 and not 10.0.30.2 — so browsing the Proxmox UI via either lab address produces a
  name mismatch in addition to the existing self-signed warning. Proper TLS is Phase 8.

### Storage pre-flight and SMART baseline (2026-08-17)

- **Both directory storages mounted despite device-letter drift:** the SSDs registered in
  Phase 2 as `/dev/sdc` and `/dev/sdd` now enumerate as `/dev/sdg1` and `/dev/sdh1` — six
  positions later — and both systemd `.mount` units held, because PVE's Disks → Directory
  wizard addresses them by `/dev/disk/by-uuid/`.
- **`is_mountpoint 1` confirmed on both** — the setting that stops PVE writing into a bare
  directory on the 69 GB root volume if a mount ever fails.
- **`ssd-inquisitor` content types narrowed** from `iso,images,rootdir,snippets,vztmpl` to
  `images` only.
- **Both SSDs PASSED**, with zero reallocated sectors, zero program/erase failures, zero
  uncorrectable errors, and zero CRC errors on each.

| Drive | Role | Power-on hours | Host writes | Life remaining |
|---|---|---|---|---|
| Crucial BX200 480 GB | `ssd-inquisitor` | 11,318 | 44.7 TiB | 94% (WAF ≈1.95×) |
| Samsung 860 EVO 500 GB | `ssd-vmstore` | 4,466 | 5.6 TiB | ~99% |

### inquisitor build and Wazuh install (TODO(cole): date)

- **Address verified before services:** 10.0.30.60 from the Kea reservation, confirmed with
  `dig` — the cantina/vault pattern, checked before anything was installed on the box.
- **Secure Boot enabled and verified**, not merely left at a default.
- **Index disk built raw:** no partition table, reserved blocks reclaimed, mounted by UUID with
  `nofail` plus a `RequiresMountsFor` drop-in, so the indexer cannot start without its storage.
- **The shadowing was confirmed forensically, not inferred:** unmounting the data disk revealed
  the package's directory underneath, untouched, with its dpkg-preserved build date intact.
  "The installer never wrote" and "something was mounted over what it wrote" present
  identically from above; only looking underneath separates them.
- **Upstream defect located by line:** the 4.14.7 installer divides a variable that is never
  assigned at line 1689, producing a 1 GB heap regardless of host RAM. Corrected to `3g`.

### wazuh-db socket outage (TODO(cole): date)

- **The symptom that exposed it:** the dashboard reported no API connection while all four
  services read `active` under systemd. The API was listening; `wazuh-db`'s Unix socket was
  gone.
- **Duration measured, not estimated:** an error every 16 seconds, for 11 days.
- **Fix tested, not assumed:** verified across two consecutive restarts with both layers in
  place — needrestart excluded, and `KillMode=control-group`.

### Memory and disk (TODO(cole): date)

- **A falsifiable prediction, stated in advance and held:** the manager's 4.2 G as reported by
  systemd was predicted to be almost entirely reclaimable page cache rather than anonymous
  memory, and was.
- **Vulnerability Detection's cost measured before it was disabled:** 13.7 GB on the root
  volume with zero agents enrolled; root usage 24 G → 11 G after reclaim.
- **executor:** 61.0 GiB allocated / 5.9 GiB swapped, tarkin the most-swapped guest at 30% →
  53.0 GiB / zero swap.
- **Not resolved:** memory PSI *worsened* after that correction — 22.2 s of full stall the
  following night against a prior average near 7 s/day. Two hypotheses with distinguishable
  signatures; a sampler is running to separate them.

### Index lifecycle (TODO(cole): date)

- **Sharding 3/day → 1/day, verified twice** — fixed at a separate order-10 template, because
  filebeat rewrites edits to its own.
- **ISM at 90 days now manages 26 time-series indices**; current-state indices are excluded
  permanently.

---

## Log

### 2026-08-16 — Proxmox alert mail had not delivered for months

#### Issue
20 messages totalling 529 KB sat deferred in postfix's queue on executor, spanning at least
2026-08-11 through 08-14 — all of them nightly vzdump reports. The 2026-08-02 pool-degraded
alert was never received at all; that drive fault was found by hand days later (Phase 5).

#### Root cause
`relayhost` was empty, so postfix did what an empty `relayhost` means: attempt **direct-to-MX**
delivery, connecting to `gmail-smtp-in.l.google.com:25`. The ISP blocks outbound TCP/25 —
standard practice, and the reason consumer connections cannot run a mail server. Confirmed by a
**paired reachability test** against `smtp.gmail.com`: **587 open, 25 blocked**. One test,
two ports, cause isolated in a single step — the discriminating test rather than the
comprehensive one.

A second defect compounded it. `inet_protocols` was at its default of `all`, so every delivery
attempt tried IPv6 first, failed with "Network is unreachable", then fell back to IPv4 and
timed out. That doubled the cost of every attempt and filled the queue with error strings
pointing at an IPv6 address — an error message that named the wrong address family and led away
from the actual fault.

#### Fix
postfix reconfigured as an **authenticated satellite relay**:

- `relayhost=[smtp.gmail.com]:587` — the square brackets suppress the MX lookup, so postfix
  connects to that host directly instead of resolving it as a mail domain.
- SASL authentication via a credentials map at mode 600, compiled with `postmap` into the
  `hash:` database postfix actually reads.
- `smtp_tls_security_level=encrypt` — **fail-closed**. The alternative, `may`, permits a silent
  fallback to cleartext, which is the wrong failure direction for a channel carrying
  credentials.
- `sender_canonical_maps` rewriting the envelope sender, because Gmail rejects a `From` that
  does not match the authenticated account.
- `inet_protocols=ipv4`.
- `libsasl2-modules` installed — without it, SASL authentication fails with a mechanism error
  that reads like a credential problem.

Queue flushed; a test message delivered with `status=sent` and `250 2.0.0 OK … gsmtp`.

#### Learning
(a) **The sending program always succeeds.** cron, smartd, and PVE hand a message to postfix
and exit. None of them waits for delivery, so a broken MTA is completely invisible from above
it — which is exactly how this stayed broken for months while every layer above it reported
success. (b) **Judge mail by the receiving server's acceptance code, not the sending command's
exit status.** That is the same discipline as reading DNS reply codes in Phase 3 and the
encoder line in Phase 4's ffmpeg log: verify the thing that actually indicates the outcome, not
the thing that indicates the attempt. (c) **IPv6-first resolution has now cost time in three
separate subsystems** — apt on cantina (Phase 4), apt on vault (Phase 5), and postfix here. The
lab is IPv4-only. Any daemon that resolves both address families needs an explicit IPv4 setting
at build time, and that now goes on the standard build checklist rather than being rediscovered
a fourth time.

### 2026-08-16 — `pvescheduler` journal noise root-caused to an unclean shutdown

#### Issue
executor logged `replication: invalid json data in
/var/lib/pve-manager/pve-replication-state.json` **every 60 seconds**.

#### Root cause
The file was **2 bytes of NUL** — `xxd` returned `0000`. Not empty, and not a malformed
configuration: a file of the correct length containing zeros. Its preserved mtime was
2026-06-06 20:12:01, which is **49 minutes before boot session -14 began** at 21:01:25.

That combination is the ext4 **delayed-allocation** signature. When a small file is written,
the inode's size update reaches the journal while the data itself is still sitting in the page
cache waiting to be flushed. If the host goes down hard in that window, recovery restores a
file of the right length filled with zeros, because the metadata was durable and the data was
not. So something wrote a valid `{}` and the host died before those two bytes reached the
platter — during Phase 2's controller-passthrough work, a period the boot list shows was full
of reboots.

#### Fix
The file was reset to `{}` with a timestamp-preserving backup taken first.
`/etc/pve/replication.cfg` is absent — no replication jobs are configured — so the operation
was lossless. Verified silent over 20 consecutive scheduler cycles.

#### Learning
(a) **Journal hygiene has to precede journal ingestion.** A 60-second error on the lab's most
important host is 1,440 lines a day; ingest that into a SIEM and it indexes noise and buries
signal, and the burial is worst during an incident when the journal is being read fastest.
(b) **`cat` cannot distinguish valid JSON from whitespace or NULs, and `xxd` can** — inspect
small configuration files as bytes when their contents are in question. (c) **A predicted
mechanism that produces the right fix can still be the wrong mechanism.** The initial theory
was stray whitespace; that theory was falsified by the hex dump, and the correction matters
because the two mechanisms have different futures — a crash artefact can recur after the next
hard reboot, while a configuration error cannot. **Falsifier:** NULs reappearing after a
Navi-10 reset-failure reboot would *confirm* the crash mechanism, not indicate that the fix
failed. Corroborating evidence for a history of unclean shutdowns is already on the drives
themselves: 137 power-off retracts on the BX200 and 62 power-on-reset recovery events on the
860 EVO.

### 2026-08-16 → 2026-08-17 — `feetwifi` purge

#### Issue
The hostname `pve.feetwifi.com` — derived from an SSID at a former residence — persisted across
`/etc/hosts`, `/etc/mailname`, the search domain in `/etc/resolv.conf`, `/etc/aliases.db`, and
the **CN and SAN of the Proxmox TLS certificate**.

#### Root cause
The FQDN entered once at install propagated into every subsystem that derives a name from it.
Nothing re-derived it later, so each subsystem kept its own copy.

#### Fix
In order, and the order mattered:

1. `/etc/hosts` retargeted to `pve.galaxy.internal` — IP and short name unchanged.
2. `postconf -e myhostname`, and `/etc/mailname` rewritten.
3. Search domain changed via the GUI.
4. `newaliases` to regenerate the alias database.
5. `pvecm updatecerts --force` to regenerate the node certificate — **after** the hosts fix,
   because the certificate derives its CN by resolving the node name. Run before step 1 it
   would have faithfully reproduced the old name.

#### Learning
(a) **A diagnostic that can print nothing cannot be trusted.** The check used was
`grep -r feetwifi /etc/ 2>/dev/null || echo CLEAN`, and it printed *nothing at all* — which
looked like success and was not. GNU grep 3.5+ sends its binary-file match notice to **stderr**
while still exiting **0**, so the discarded notice plus a zero exit meant the `|| echo CLEAN`
branch never ran. The signal was the **absence of the word CLEAN**, which is precisely the kind
of signal a human does not notice. Use `grep -rq PATTERN /etc/ && echo FOUND || echo CLEAN`
instead, so exactly one of two words always prints and silence becomes impossible.
(b) **The TLS certificate was the one real exposure on the list.** It is presented to every
browser that connects to `:8006`, and no text search of the repo would ever have surfaced it —
the same class of finding as the rasterised diagram in the 2026-08-16 exposure audit (Phase 5):
the thing grep cannot see is the thing worth looking for by hand. (c) **Node renaming was
deliberately not attempted.** The short name `pve` stays. Renaming a PVE node means relocating
`/etc/pve/nodes/pve/` and fixing storage and job references, for no functional gain — and the
Wazuh agent will simply register as `executor`, since agent names are arbitrary and set at
enrolment (ADR 0020).

**Residual, accepted:** the regenerated certificate's SAN covers only 192.168.1.225 — not
10.0.10.10 and not 10.0.30.2 — so browsing the UI via either lab address produces a name
mismatch as well as the self-signed warning. Proper TLS is Phase 8.

### 2026-08-17 — Storage pre-flight

Device letters had drifted. The two SSDs registered in Phase 2 as `/dev/sdc` and `/dev/sdd` now
enumerate as `/dev/sdg1` and `/dev/sdh1` — **six positions later**. Both mounts survived without
any intervention, because the systemd `.mount` units generated by PVE's Disks → Directory
wizard address the filesystems by `/dev/disk/by-uuid/` rather than by device node.

Working hypothesis for the six consumed letters, **~80% confidence**: the host briefly
enumerates the ASM1064's six SATA drives at boot, before `vfio-pci` claims the controller for
passthrough. That is consistent with the note already in
[runbooks/pcie-passthrough.md](../runbooks/pcie-passthrough.md) that the drives stay visible on
the host until the VM starts.

Also confirmed in the same pass: **`is_mountpoint 1`** on both directory storages — the setting
that stops PVE silently writing into a bare directory on the 69 GB root volume if a mount ever
fails — and `ssd-inquisitor`'s content types narrowed from
`iso,images,rootdir,snippets,vztmpl` down to **`images`** only, ahead of the 200 GiB data disk
landing there.

#### Learning
This is the **second empirical confirmation in this lab** that device letters drift and UUIDs
do not — the first being echo-base's pool, deliberately created against `/dev/disk/by-id` in
Phase 5. Record it as evidence rather than as a hypothetical argument: the reason to address
disks by stable identifier is no longer "best practice says so", it is "it has now happened
twice here and both times the UUID-addressed configuration was the one that survived."

### 2026-08-17 — SMART baseline; `ssd-inquisitor` stays on the weaker drive

Both SSDs **PASSED** self-assessment with zero reallocated sectors, zero program/erase
failures, zero uncorrectable errors, and zero CRC errors.

- **Crucial BX200 480 GB** (`ssd-inquisitor`) — 11,318 power-on hours, 44.7 TiB host writes,
  94% life remaining, write amplification ≈**1.95×** measured from the TLC/SLC write counters.
- **Samsung 860 EVO 500 GB** (`ssd-vmstore`) — 4,466 hours, 5.6 TiB written, ~99% remaining.

**Decision: no swap.** The BX200 is the more worn drive and has a known weakness in sustained
steady-state writes, which raises the obvious question of whether the SIEM's data disk belongs
on the Samsung instead. Three arguments say no. Wazuh at this scale writes roughly **200 MB/day
against ~68 TB of remaining endurance**, so the weakness never binds — it is a trickle, not a
sustained stream. The genuinely heavy writer is the **nightly vzdump job**, which is already on
the better drive and would have to move onto the BX200 to make room. And the *value* of the two
datasets is opposite: [ADR 0019](../decisions/0019-siem-data-placement-and-backup.md) declares
index data expendable, while [ADR 0009](../decisions/0009-infrastructure-vm-backups-local.md)
makes `ssd-vmstore` the only local copy of the router and NAS configurations. The more valuable
data is already on the better drive.

#### Learning
The important finding is about the **instrumentation**, not the wear. **Every SMART threshold
on the BX200 is `000`** except attribute 169, and the normalized VALUE column is frozen at 100
while 169's raw value has already moved to 94. A drive whose thresholds are all zero has a
self-assessment that is structurally incapable of failing: normalized value can never drop
below a threshold of zero, so any monitoring that reads normalized values will report a perfect
disk indefinitely, right up to the moment the drive stops answering. The Samsung, by contrast,
carries real Pre-fail thresholds of `010`.

**Consequence:** the Phase 6 SMART rule for `ssd-inquisitor` must read **raw** attribute values.
This is the same lesson as Phase 4's GPU counters — know what a counter measures before
trusting its reading, and specifically before trusting a zero.

**Replacement trigger recorded:** attribute 169 raw value below **80**, or **any nonzero value**
in attributes 5, 181, 182, or 199.

### TODO(cole): date — inquisitor built; Wazuh 4.14.7 all-in-one installed

VM 105 built to the ADR 0018 specification — 3 vCPU with CPU units 50, 8 GB with ballooning
off, Ubuntu Server 24.04 standard install, vmbr1 tag 30, and 10.0.30.60 from a Kea reservation,
`dig`-verified before anything was installed on the box.

Two deliberate departures from the written spec, both reasoned through under Decisions &
Rationale above. **Secure Boot is ON** — the passthrough runbook's no-Secure-Boot instruction
is TrueNAS-specific and does not apply to signed Ubuntu. And the **200 GB index disk is raw**:
no partition table, reserved blocks reclaimed, mounted by UUID with `nofail` and a
`RequiresMountsFor` drop-in so the indexer cannot start without its storage. `backup=0` per
ADR 0019.

The installer was read before it was run, which is how the heap defect two entries below was
found.

### TODO(cole): date — The Wazuh installer failed twice, both times self-inflicted

#### Issue
Attempt 1 refused to run at all, reporting Wazuh already installed. Attempt 2 installed but
failed at first start — OpenSearch could not create `nodes/`.

#### Root cause
Both failures came from the index disk having been prepared *before* the installer ran.

Attempt 1: the installer's "already installed" check is a **bare directory test**. The
mountpoint staged for the index disk existed and was empty, which satisfied that test with no
Wazuh packages present anywhere on the system.

Attempt 2: `RequiresMountsFor` **implies `Requires`**, not merely `After`. systemd therefore
*activated* the mount unit at service start, and the mount landed on top of the directory the
installer had populated minutes earlier. OpenSearch opened an empty, root-owned filesystem and
could not create `nodes/` inside it.

The second diagnosis was confirmed forensically rather than reasoned to: unmounting the data
disk revealed the package's directory underneath, untouched, with its dpkg-preserved build date
intact. Shadowing and never-writing look identical from above.

#### Fix
Let the installer install stock, confirm the stack healthy, *then* move the data directory onto
the index disk and re-apply the mount ordering.

#### Learning
**Configuration staged for software that does not yet exist cannot be validated against
anything.** Preparation ahead of an opinionated vendor installer looks like diligence and
behaves like a defect, because its mistakes surface as installer failures two layers from their
cause. Install stock, confirm, then adapt. The corollary, from the half of this that worked:
reading the installer first is cheap, and in the same sitting it found a real upstream bug.

### TODO(cole): date — Wazuh 4.14.7 sets a 1 GB indexer heap regardless of host RAM

#### Issue
The installer configures the indexer heap at 1 GB on an 8 GB VM, ignoring available memory.

#### Root cause
An upstream defect, found by reading the script rather than by observing symptoms: **line 1689
divides a variable that is never assigned.** The intended behaviour is to size the heap from
detected RAM; the unset operand collapses the computation to a floor value.

#### Fix
Heap set explicitly to `Xms=Xmx=3g`, the ADR 0018 figure — under half the VM's 8 GB, leaving
the remainder to the OS page cache that Lucene's memory-mapped segments are actually read
through.

#### Learning
This is exactly the failure ADR 0018's sizing section exists to prevent, arriving from a
direction the ADR did not anticipate: not an operator setting the heap wrong, but a vendor
installer silently setting it wrong and reporting success. It would have presented much later
as an unexplained performance ceiling with no error anywhere in the logs. Reading an installer
costs minutes, and paid for itself immediately here.

### TODO(cole): date — Eleven days of silent failure: the wazuh-db socket

#### Issue
The Wazuh dashboard reported no API connection. All four services read `active` under systemd.

#### Root cause
A chain, each link individually reasonable:

1. `unattended-upgrades` installed a new **libc6**.
2. `needrestart` noticed and restarted the Wazuh manager — correct behaviour after a libc
   update.
3. The vendor unit file sets **`KillMode=process`**, so systemd signalled only the main process
   and left its children running.
4. `wazuh-db` unlinked its Unix socket on the way down, then hung and **survived the kill**.
5. `wazuh-control` subsequently skipped it as "already running" — which it was, in the sense of
   holding a live PID, and useless, in the sense of having no socket.

Every dependent component then failed against a socket that no longer existed, logging an error
**every 16 seconds for 11 days**, while systemd reported four healthy services throughout.

#### Fix
Two layers, because either alone leaves the failure reachable:

- **needrestart no longer auto-restarts Wazuh** — the trigger is removed.
- **A `KillMode=control-group` drop-in** — a stop now stops the whole cgroup, so no straggler
  survives to be mistaken for a healthy process.

Verified across two consecutive restarts.

#### Correction
The first prevention plan was `apt-mark hold` on the Wazuh packages, and it was wrong. The
trigger was **libc6**, not a Wazuh package — holding Wazuh would have prevented nothing while
freezing the SIEM's own patch cadence as a side effect. Recorded because the wrong fix was
plausible enough to have shipped.

#### Learning
(a) **`active` means "the main PID is alive", not "the service works."** This is the third
incident in this phase of a system reporting healthy while the path through it was dead: mail
that always "sent", a grep that reported clean by printing nothing, and now a service that is
up without being usable. The remedy is identical every time — assert on the thing that carries
the outcome, not the thing that reports the attempt. Here that means a health check which
opens the socket.
(b) **`KillMode=process` is a liability for any multi-process daemon.** It turns an ordinary
restart into a partial one, and a partial restart into a permanent half-state.
(c) An SIEM that cannot detect its own failure is the sharpest available argument for the
deadman in ADR 0021, which is still unbuilt. Eleven days is precisely the gap a 90-minute
grace period closes.

### TODO(cole): date — Vulnerability Detection filled 13.7 GB of root before a single agent existed

#### Issue
inquisitor's root volume stood at 24 G used. No agents were enrolled; there was nothing to
scan.

#### Root cause
Vulnerability Detection is **on by default** in 4.14.7 and refreshes its CVE feed hourly. The
feed is downloaded and indexed whether or not any agent inventory exists to evaluate against
it, so the cost is incurred by the feature being *enabled*, not by it being *used*.

#### Fix
Disabled; storage reclaimed. Root went from 24 G to 11 G.

#### Learning
ADR 0018 deferred vulnerability detection "until after soak" on an unstated premise — that
enabling it would be an opt-in act. It is opt-*out*, and the default costs 13.7 GB on a 40 GB
system disk before delivering anything at all. The decision survives; the premise did not. When
it is eventually enabled after soak, the disk headroom has to be planned rather than
discovered.

Recorded alongside it because it was stated in advance and held: the manager's 4.2 G as
reported by systemd was predicted to be almost entirely reclaimable page cache rather than
anonymous memory, and was. Stating a falsifiable expectation before measuring is what turns a
measurement into a test.

### TODO(cole): date — net-tools missing broke two detections; only one of them said so

#### Issue
Two Wazuh checks were not working: rootcheck's port check, and rule 533's port-change
monitoring.

#### Root cause
`net-tools` was absent — the Ubuntu 24.04 standard install no longer ships `netstat`, and both
checks shell out to it.

The two failures behaved completely differently. **rootcheck logged a warning.** **Rule 533
failed silently, and had done since install:** it works by comparing the current port list
against the previous one, and with the command missing it compared **empty output against
empty output**, found no difference, and reported no change — indistinguishable from a
healthy, stable system.

#### Fix
`net-tools` installed.

#### Learning
The silent one is the one that matters. A detection whose input has disappeared does not
necessarily fail: a differential check with no data on *both* sides succeeds, permanently and
quietly, and "no change" is exactly the output you were hoping for. Any check built on a diff
needs a liveness assertion on its **input**, not only on its output. Same species as the grep
that reported clean by printing nothing, earlier in this phase.

### TODO(cole): date — executor overcommitted; the router was the most-swapped guest

#### Issue
executor was allocating **61.0 GiB** against 62.5 GiB usable, with **5.9 GiB swapped**.

#### Root cause
Accumulated allocation with no budget enforced at creation time. The detail that matters is
*which* guest paid: the most-swapped VM, at **30%**, was **tarkin — the router**. That is the
predictable outcome rather than bad luck. Idle infrastructure has the coldest pages and the
kernel evicts by coldness, so the guests whose stalls are most damaging are exactly the ones
most likely to be paged out. A router that stalls takes routing, DHCP, and inter-VLAN traffic
with it.

#### Fix
Allocation reduced to **53.0 GiB**; swap drained to zero. **New guardrail:** every new VM is
checked against the host memory budget *before* creation.

#### Correction
An earlier working claim in this phase — that the indexer's `bootstrap.memory_lock` protects
its heap — is wrong at the host layer, and the correction matters because it reassigns
responsibility for the problem. `memory_lock` pins the heap within the guest's address space
and stops the *guest* kernel swapping it. To executor, inquisitor is one 8 GB QEMU process,
and the host may page out any of it irrespective of guest-side locking. Only PCI passthrough
pinning (ADR 0018's context) or not overcommitting reaches that layer.

#### Unresolved
The fix did not produce the predicted improvement — memory PSI got **worse**. Full-stall time
the following night was **22.2 s**, against a prior average near **7 s/day**. Two hypotheses
with distinguishable signatures are open, and a sampler is running to separate them. Recorded
as open rather than closed: reporting the intervention without the contradicting measurement
would make this write-up wrong. Carried in Deferred below.

### TODO(cole): date — Oversharding fixed at the template layer

#### Issue
The indexer was creating **3 shards per day** where 1 is correct for a single-node cluster at
this volume. Excess shards cost heap and file handles and buy nothing without a second node.

#### Root cause
Shard count comes from the index template, and editing **filebeat's own template** does not
hold — filebeat rewrites it.

#### Fix
A **separate order-10 template**, which takes precedence and survives filebeat rewriting its
own. 3 shards/day → 1, verified twice.

#### Lifecycle, settled in the same pass
ISM retention at 90 days now manages **26 time-series indices**. **Current-state indices are
excluded permanently** — they hold live inventory rather than history, so age-based deletion
there would destroy the current picture instead of old data. That distinction is a property of
what the index *is*, not a tuning preference, which is why the exclusion is permanent rather
than provisional.

#### Learning
When a vendor component owns a configuration object, do not edit that object — add a
higher-precedence one beside it. The edit-in-place version appears to work until the component
next writes, and then reverts with no error and no event to notice.

### 2026-09-26 → 2026-09-27 — ISP DNS interception broke recursion, and only recursion

#### Issue
Names began failing — but one at a time, over several hours, rather than all at once. The lab
stayed usable throughout, which is most of why this took as long as it did to characterise.

#### Root cause
The ISP began **transparently intercepting all outbound port 53 traffic**, following a
same-day plan change on the account.

Proven in a single step: a query for a name pointed at an **RFC 5737 TEST-NET-1** address
returned **NOERROR with answers, over both UDP and TCP**. No server exists at that address and
by standard none can — the range is reserved for documentation and is unroutable. Anything
that answers is therefore not the server that was addressed. That is a positive proof of
interception that needs no baseline, no packet capture, and no cooperation from the ISP.

**Why it broke recursion and nothing else.** Unbound in recursive mode asks *authoritative*
servers *authoritative* questions and expects non-recursive referrals back — that is how a
delegation walk works, one referral at a time from the root down. The interceptor answered
every query **recursively**. Unbound correctly classified each responder as **REC_LAME** — a
server offering recursion where a referral was due — and discarded it, so the delegation walk
could never complete.

Meanwhile every **forwarder** on the network kept working and reported healthy: Pi-hole,
phones, laptops, IoT. A forwarder *wants* a recursive answer and got exactly that. The
interceptor was, from their point of view, a perfectly good resolver.

**The shape of the failure was cache expiry, not degradation.** Nothing broke when the
interception started. Names failed individually as their TTLs aged out of Unbound's cache,
which is why the onset looked gradual and arbitrary rather than like a single event.

#### Diagnostic path — two falsified hypotheses, and an hour spent looking past the answer

Recorded in full rather than tidied, because the failures here are more instructive than the
fix.

- **A false correlation was adopted early.** The fault surfaced shortly after an unrelated
  `swapoff` on another host, and that timing was initially treated as causal. It was
  coincidence. This is the **second false correlation this phase**, and the same lesson as
  Phase 5's 2026-07-28 outage versus the 2026-08-02 drive fault: establish event ordering
  from timestamps, and remember that adjacency is not evidence.
- **The first test was incapable of producing the evidence it was run for.** The domain chosen
  to test DNSSEC validation was **unsigned**, so it could not have exhibited the DNSSEC
  failure it was selected to detect. A test that cannot fail cannot pass either — it returns a
  clean result that means nothing, which is worse than no test, because it gets believed.
- **The decisive evidence was in hand an hour before it was recognised.** A reply from a
  **root server** carried the **`ra` flag and lacked `aa`**. Root servers never offer
  recursion and always answer authoritatively for the root zone, so `ra` without `aa` from a
  root server is, on its own, conclusive proof that something other than a root server
  answered. **The status code was read; the flags were not.**

#### Fix — decided, not yet implemented
Plaintext recursion is impossible on this connection: there is no configuration of Unbound
that survives an interceptor rewriting every port 53 answer. The resolver therefore has to
move to **DNS over TLS forwarding** on port 853, which the interception does not touch because
it is neither port 53 nor plaintext.

**DoT verified clean on 853** against both candidate upstreams, with **CA-chained
certificates** confirmed — not merely "a connection succeeded", since an intercepted TLS
session that could not present a valid chain is exactly what this test is for.

Implementation is **blocked pending an ADR**. Moving from private recursion to third-party
forwarding reverses a decision ADR 0004 and network/dns-design.md make deliberately — "no
public resolver sees the lab's query stream" — and that reversal is a cross-cutting design
change, not a configuration fix.

#### Two deviations live and tracked
Both are interim, both are departures from the documented design, and both are recorded in
[network/dns-design.md](../network/dns-design.md) rather than left as undocumented drift:

1. **Pi-hole's upstream** is not Unbound-on-tarkin as ADR 0004 and the DNS design specify.
2. **DNSSEC validation is off.** This one carries a **hard deadline: the 2026-10-11 root KSK
   rollover.** It is a date on the calendar rather than a preference, and it is the binding
   constraint on how long the DoT ADR can sit unwritten.

#### Learning
(a) **Fourth instance this phase of a system reporting healthy while the path through it was
dead** — after mail that always "sent", a grep that reported clean by printing nothing, and a
service that was `active` without a socket. What is new here is *where* the healthy reports
came from: the **majority** of the system. Every forwarder in the lab worked perfectly. Only
the single component asking a different *kind* of question failed. That inverts the usual
troubleshooting heuristic — "many things are broken, find the common cause" — into "one thing
is broken, and it is the only one asking honestly."

(b) **Read the flags, not just the status.** `NOERROR` says the transaction completed. `aa`
and `ra` say *who* answered and *how*. The evidence was sitting in a reply an hour before it
was recognised, because the status line was read and the flag line was not. This is the same
discipline as Phase 3's DNS reply codes, Phase 4's ffmpeg encoder line, and this phase's
"judge mail by the server's acceptance code" — with the refinement that the field carrying the
answer is not always the field labelled "result".

(c) **A test that cannot fail cannot pass.** Selecting an unsigned domain to test DNSSEC
produced a clean result that meant nothing. Before running a diagnostic, confirm it is capable
of exhibiting the failure being looked for — the same defect class as rule 533 comparing empty
output against empty output, two entries above.

(d) **Temporal adjacency is not evidence.** Second false correlation this phase.

(e) **The RFC 5737 probe is worth keeping as a technique.** Query something at an address that
*cannot* host a server — TEST-NET-1, TEST-NET-2, TEST-NET-3. Any answer at all is proof of
interception. It requires no baseline, no prior capture, and no trust in the network being
tested, and it works the same way for any protocol an operator might silently proxy.

---

## Deferred / follow-ups

### Open investigations

- **ISP DNS interception — DoT migration ADR unwritten.** Plaintext recursion is impossible on
  this connection (log, 2026-09-26/27). DoT on 853 is verified working against both candidate
  upstreams, but moving from private recursion to third-party forwarding reverses ADR 0004 and
  needs its own ADR before implementation. **Two deviations are live in the meantime** —
  Pi-hole's upstream, and DNSSEC validation, the latter against a hard **2026-10-11** root KSK
  rollover deadline.
- **Memory PSI regression on executor — unresolved.** Reducing allocation from 61.0 GiB to
  53.0 GiB and draining swap to zero made memory pressure *worse*, not better: 22.2 s of full
  stall the following night against a prior average near 7 s/day. Two hypotheses with
  distinguishable signatures are open and a sampler is running to separate them. The overcommit
  fix is **not** closed until this resolves — the intervention and the measurement currently
  disagree, and the measurement wins.
- **No self-health check on the SIEM yet.** The wazuh-db outage ran 11 days because `active`
  under systemd does not mean the service works. A check that opens the socket, plus the
  ADR 0021 deadman, are what close this; both are unbuilt.

### Accepted conditions

- **No back-history to tune against.** The interim syslog sink planned in
  docs/telemetry-logging.md was never built, so Phase 6 starts with zero historical telemetry.
  Accepted, not scheduled: rules are tuned against deliberately triggered events using
  `wazuh-logtest` against captured real log lines. Frequency- and baseline-derived rules wait
  for the lab to generate their own history.
- **executor is invisible to Pi-hole.** Its resolver is senate by design (above), so host DNS
  visibility comes from journald ingestion instead. Recorded in
  [network/dns-design.md](../network/dns-design.md) and
  [docs/telemetry-logging.md](../docs/telemetry-logging.md) so it is a stated decision rather
  than undocumented drift.
- **death-star is an unmonitored source.** The switch manual's chapter structure shows no
  obvious remote-syslog section, so it is not yet established that it can export anywhere.
  Confirm the capability in the switch UI before planning around it (ADR 0020).
- **Proxmox UI certificate name mismatch on lab addresses.** The regenerated SAN covers
  192.168.1.225 only. Phase 8.

### Carried in from Phase 5

- **Immich DB dumps are unmonitored** (Phase 5 item 2) — becomes a Wazuh rule on
  no-new-dump-in-48h, configured **locally on vault** because `remote_commands` is disabled by
  design (ADR 0020).
- **Immich patch cadence needs a deliberate update process** (Phase 5 item 3) — still open; not
  a monitoring task, but it belongs to this phase's backlog.
- **`holocron/configs` does not exist** (Phase 5 item 6) — now scheduled under
  [ADR 0019](../decisions/0019-siem-data-placement-and-backup.md), which puts
  `/var/ossec/etc/` on it nightly and adds it to the syncoid set. Creating it also finally gives
  the OPNsense XML export and the switch config an offsite copy.
- **Cold spare drive policy** (Phase 5 item 7) — and note the gap this phase adds: the policy as
  written covers **4 TB HDDs only**. There is no spare SSD, which now matters, because the
  SIEM's data disk sits on the lab's most-worn drive.

### Deferred to Phase 7

- **Sysmon on falcon** — process-creation telemetry is worth collecting once scout exists to
  generate events worth detecting (ADR 0020).
- **Active response** — currently disabled globally; auto-blocking in a lab whose router is a VM
  on the same host as the SIEM is a self-inflicted lockout waiting to happen. The attack range
  is the safe place to test it (ADR 0020).
- **OpenSearch snapshot repository** — index data is deliberately not backed up. Revisit when
  attack-exercise detection history becomes a deliverable rather than an operational
  convenience (ADR 0019).

### Deferred to soak

- **Vulnerability detection** — the heaviest single feature in the product, and **on by
  default**, which ADR 0018 did not anticipate: it put 13.7 GB on the root volume before a
  single agent was enrolled and has been disabled (log above). Re-enable only after the
  deployment has soaked, so a resource problem can be attributed to the feature or to the
  sizing rather than to both at once — and plan the disk headroom in advance this time rather
  than discovering it.
