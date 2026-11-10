# 0018 — Wazuh deployment model and sizing: all-in-one on inquisitor
**Status:** Accepted (Phase 6, 2026-08-17)

**Context:** The SIEM has to fit inside a host that is already carrying the lab. executor is
an i7-7700T — 4 physical cores, 8 threads — with 62.5 GiB usable RAM, and it already runs the
router, the NAS, a media server, and a photo platform. Measured allocation before inquisitor
exists is 49.0 GiB across seven VMs, leaving roughly 15 GiB genuinely available.

The memory column understates the constraint. archives (16 GiB) and cantina (6 GiB) both use
PCI passthrough, and a passthrough VM's *entire* guest memory is pinned: the assigned device
performs DMA directly against guest-physical addresses with no hypervisor in the path to fix
up a translation, so those pages cannot be swapped, ballooned, or KSM-merged. Counting QEMU's
per-VM overhead, ~22.5 GiB of the measured 49.0 GiB is immovable by construction — a cost that
is invisible if you read the allocation table and assume the usual overcommit levers apply.
Wazuh's own footprint therefore has to be right the first time; there is no slack to borrow.

**Decision:** A single all-in-one Wazuh 4.14.x stable deployment — manager, indexer, and
dashboard on one VM (inquisitor, VM 105, 10.0.30.60). Sizing and configuration as built:

| Setting | Value |
|---|---|
| CPU | 3 vCPU, type `host`, CPU units 50 |
| RAM | 8192 MiB, ballooning OFF |
| Indexer heap | `Xms=Xmx=3g` |
| System disk | 40 GiB on `local-lvm` (NVMe) |
| Data disk | 200 GiB raw on `ssd-inquisitor`, **Backup unchecked** (`backup=0`) |
| Kernel | `vm.max_map_count=262144` |
| Index replicas | `number_of_replicas: 0` |
| Archives indices | disabled |
| Retention | ISM policy — rollover, delete at 90 days |
| Vulnerability detection | off initially; enabled after soak |
| OPNsense logs | block logs only, not pass logs |

**Alternatives:**

*Distributed deployment* (rejected) — separate manager, indexer, and dashboard nodes multiply
JVM and OS overhead across the same silicon for no gain, and the high availability such a
layout exists to provide is meaningless here: every node would share one host, one PSU, and
one set of four cores. Paying a real resource cost for a property the physical layer cannot
deliver is a worse outcome than not having it.

*Wazuh in Docker on shipyard* (rejected, most strongly) — shipyard holds the lab's only
inbound WAN port forward (Minecraft, 25565). Co-locating the SIEM with the most externally
exposed host in the lab means a successful compromise lands directly on the evidence store,
with the ability to delete the alerts describing its own arrival. The monitoring system must
not share a host with the thing most likely to be attacked.

*Security Onion* (rejected) — it replaced Wazuh with Elastic Agent in its 2025 release, so it
no longer teaches the agent/decoder/rule model this phase is built around, and its manager
alone wants 16 GB+, which is more than the entire remaining headroom.

*Plain OpenSearch or ELK* (rejected) — everything Wazuh supplies above the index (agent
enrolment and management, decoders, rule evaluation, FIM, SCA) would have to be hand-built.
That is a bigger project than the phase, and the hand-built version would be worse.

*Wazuh 5.0 beta* (rejected) — a monitoring system is the worst possible place to run beta
software, because when it breaks you simultaneously lose the ability to observe that it broke.
Revisit trigger: 5.0 GA plus at least one patch release.

*The official Wazuh OVA* (rejected) — ships on Amazon Linux, off-pattern for a lab that is
Debian and Ubuntu everywhere else, which would mean a second package manager, a second update
discipline, and a second set of paths to remember during an incident.

**Consequences:**

**Heap never exceeds 50% of VM RAM, and that is not a rule of thumb.** Lucene — the index
engine underneath OpenSearch — memory-maps its immutable segment files and relies on the
operating system's page cache to serve them. RAM given to the JVM heap is RAM taken away from
that cache, so pushing the heap up past half of system memory makes search *slower*, not
faster: the heap grows, the cache shrinks, and reads that were being served from RAM go back
to the SSD. 3 GiB of heap against 8 GiB of VM memory leaves the kernel roughly 4.5 GiB to
cache segments with. `Xms=Xmx` (identical minimum and maximum) is set so the JVM allocates its
full heap at startup and never resizes it, which removes a class of stop-the-world pause that
otherwise arrives exactly when ingest is heaviest.

**`vm.max_map_count=262144` is a prerequisite, not tuning.** Every memory-mapped segment
consumes one kernel memory-map entry, and the Linux default is about 65,530 for the entire
process. An index with many shards and segments exhausts that ceiling and the node fails to
start or falls over under load, with an error that reads like an OpenSearch problem and is
actually a kernel limit. It is set persistently in `/etc/sysctl.d/`, not at runtime.

**`number_of_replicas: 0` because a single-node cluster cannot allocate a replica.** A replica
shard must live on a *different* node than its primary; with one node there is nowhere to put
it, so the cluster reports yellow permanently. That is worse than having no health indicator
at all — a signal that is always yellow trains you to stop reading it, and then the day it
turns yellow for a real reason it looks exactly like every other day. Setting replicas to 0
makes green mean green.

**ISM exists to prevent the flood-stage watermark, not to save disk.** OpenSearch has three
disk watermarks: at 85% it stops allocating new shards to the node, at 90% it tries to
relocate shards away, and at 95% — flood stage — it flips every index on the node to
`read-only-allow-delete`. At that point the SIEM stops recording while the process is still
running, the dashboard still loads, and the agents still connect. It looks alive and it is
deaf. The ISM policy (rollover, then delete at 90 days) keeps usage away from that cliff so
the failure never has to be recovered from.

**Archives indices stay disabled** because they store *every* event the manager receives
rather than only the events that generated an alert. On a lab with this many sources that
fills 200 GiB in weeks, and the events it preserves are the ones no rule found interesting.

**200 GiB on a ~417 GiB usable volume leaves roughly half the drive unmapped**, which is
deliberate. Unwritten LBAs remain available to the SSD controller as spare blocks for wear
levelling and garbage collection, and that headroom matters more on `ssd-inquisitor` than it
would elsewhere, because that drive has a known weakness in sustained steady-state writes
(see docs/hardware-inventory.md and ADR 0019).

**CPU units 50 is a weight, not a cap.** It is a cgroup share applied only when the host is
actually contended: on an idle host inquisitor bursts to whatever it needs, and when tarkin
is routing under load inquisitor yields first. Indexing is deferrable; routing is not. This is
the same reasoning applied to vault in Phase 5.

**Vulnerability detection is deliberately deferred** until the deployment has soaked. It is
the heaviest single feature in the product — it pulls CVE feeds and scans every agent's
package inventory — and enabling it on day one would make it impossible to tell whether a
resource problem came from the feature or from the sizing above.

**OPNsense block logs only.** Pass logs on a busy interface are the highest-volume, lowest-
information source available; block logs are the ones that carry the lateral-movement signal
this lab was segmented to produce (docs/telemetry-logging.md).

## Amendment — 2026-11-10 (as built: three premises that did not hold)

The decision above stands. What follows records where the build diverged from the text, so the
table is not read as a description of the running system. Evidence for each item is in
[phases/phase-6-inquisitor-wazuh.md](../phases/phase-6-inquisitor-wazuh.md).

**1. The memory figure in Context was a snapshot, and was read as a budget.** "49.0 GiB across
seven VMs, roughly 15 GiB available" was accurate on the day it was measured. By 2026-09-24
allocation had reached 61.0 GiB across nine guests with 5.9 GiB in swap: inquisitor's 8 GiB,
sized against the figure, and a coursework VM's 4 GiB, added without consulting it. The defect
was not the number but presenting a point-in-time measurement as a standing property. Corrected
to 53.0 GiB with zero swap, and the check now happens before a VM is created
([docs/conventions.md](../docs/conventions.md)). The passthrough paragraph was confirmed by
measurement — the pinned memory shows as locked (`VmLck`) for the full guest allocation — with
one refinement: it is the guest's RAM that is pinned, not the whole QEMU process.

**2. "Vulnerability detection: off initially" described a state the product does not start
in.** It is enabled by default in Wazuh 4.14 and has to be disabled deliberately. The decision
did not say so, nothing in the build did it, and the feature wrote 13.7 GB of CVE feed to the
root volume before a single agent was enrolled. Disabling it is a **required build step**.
Re-enabling stays deferred to after soak, with one added condition: its feed is bulk
regenerable data and is planned off the root volume first (the ADR 0019 rule).

**3. The 3 GiB heap was specified as if the installer would honour system RAM.** It does not.
`wazuh-install.sh` 4.14.7 writes a 1 GB heap regardless of host memory, because line 1689
divides a variable that is never assigned, and it reports success. Setting the heap is a
**required post-install step with its own verification** (`jvm.options` and the running JVM's
arguments), not a tuning choice. One consequence the text above does not state:
`MaxDirectMemorySize` scales to half the heap, so 3 GiB of heap reserves about 4.5 GiB of an
8 GiB VM. The heap cannot be raised on this host without raising VM RAM first, and item 1 is
what blocks that.

The common thread: each of the three recorded an intended end state without recording what the
system does by default, so nothing in the build would have caught the gap.

**Settings in the table that were built differently, or needed no building:**

| Setting | As decided | As built |
|---|---|---|
| Kernel | `vm.max_map_count=262144` | `1048576`. Ubuntu 24.04 already ships that value; setting 262144 **lowered** it and was corrected |
| Index replicas | `number_of_replicas: 0` | Already the Wazuh 4.14 default — verified, not configured |
| Index shards | not addressed | The default is 3 primary shards per daily index. Reduced to 1 with a separate order-10 template (`wazuh-custom`), because filebeat rewrites edits to its own |
| Retention | ISM policy — rollover, delete at 90 days | `wazuh-retention-90d`: delete at 90 days, **no rollover** — daily and weekly indices are already period-bounded |

**Retention scope, stated because getting it wrong destroys data:** the policy covers
`wazuh-alerts-4.x-*`, `wazuh-archives-4.x-*`, `wazuh-statistics-*` and `wazuh-monitoring-*`.
`wazuh-states-*` indices are excluded permanently. They carry no date suffix — each is a single
current-state index updated in place — so deleting one by age would remove every host's live
inventory, not old data.
