# 0019 — SIEM data placement and backup: local disk, configs replicated, indices expendable
**Status:** Accepted (Phase 6, 2026-08-17)

**Context:** Wazuh's indexer has the same storage requirements as any Lucene-backed engine,
and the lab's default instinct — put bulk data on holocron over NFS — is wrong here for the
same reasons it was wrong for Immich's database in ADR 0015. Separately, the question of what
an SIEM backup is actually *for* has to be answered before deciding what to back up: alerts
are derived evidence, but agent keys and manager configuration are not.

**Decision:**

1. Indexer data lives on a **dedicated local virtual disk** on `ssd-inquisitor` — never on
   NFS, and not on the system disk.
2. Create the dataset **`holocron/configs`** and export `/var/ossec/etc/` to it nightly,
   adding that dataset to the syncoid replication set (ADR 0010).
3. **No OpenSearch snapshot repository** is registered.
4. The index data disk is **excluded from vzdump** (per-disk Backup checkbox cleared,
   `backup=0`).

**Alternatives:**

*Index data on holocron over NFS* (rejected) — Lucene needs low-latency random IO, it
memory-maps its segment files, and it depends on POSIX file locking to keep two processes from
writing one index. NFS provides all three badly and upstream does not support network
filesystems for data paths. The decisive argument is not performance, though: an NFS stall
parks the JVM in uninterruptible IO, which is the exact deadlock class ADR 0009 was written
about after the 2026-06-23 incident. This is ADR 0015's Postgres reasoning applied to a second
engine, and the fact that two independent applications produced the same answer is the point —
it is a property of the storage layer, not of Immich.

*A larger root disk instead of a separate data disk* (rejected) — the failure modes are not
comparable. A full root filesystem breaks journald, sshd session setup, and apt, so the host
becomes hard to log into at precisely the moment you need to log into it. A full *data* disk
stops the indexer and nothing else, and it announces itself cleanly through the flood-stage
watermark described in ADR 0018. Separating the disks converts a host-level failure into a
service-level one with a built-in signal.

*Registering an OpenSearch snapshot repository* (rejected for now) — it would need a
`path.repo` on shared storage, which means either putting a snapshot target on NFS or carving
more local disk, and the thing it protects (index contents) is the thing this ADR declares
expendable. Revisit trigger below.

**Consequences:**

**`client.keys` is a secret and is treated as one.** `/var/ossec/etc/client.keys` holds the
pre-shared authentication key for every enrolled agent (ADR 0020). It goes to
`holocron/configs` and never into this repo, under any circumstances. carbonite — the offsite
pool — is ZFS-encrypted at rest with a key that never leaves echo-base (ADR 0010), which makes
it the correct destination for exactly this kind of material: a stolen appliance yields
ciphertext.

**Creating `holocron/configs` closes Phase 5 deferred item 6** and finally gives the OPNsense
XML export and the death-star switch configuration an offsite copy. Those two files have lived
only in the local, gitignored `config-backups/` folder since Phase 2 — a gap ADR 0010's
Consequences listed as a requirement and left unmet, and one recorded in docs/threat-model.md
as a hole in design goal 4 (recoverability). A total site loss currently means rebuilding
tarkin and death-star from runbooks rather than restoring them; after this, it does not.

**Index data is not separately backed up, and that is an accepted risk stated plainly.** vzdump
covers the VM itself — the manager, its rules, its decoders, and its configuration — so a
rebuild is a restore plus a resync. What is not covered is the alert history in the indexer.
That is acceptable because alerts are *derived* evidence: they are the product of rules run
against logs, and the source logs still exist on the endpoints that produced them. Losing the
index costs history and dashboards, not ground truth.

**Revisit trigger:** Phase 7. The moment attack-exercise detections need to be preserved beyond
the retention window — because the detection history has itself become a deliverable rather
than an operational convenience — this decision reopens, a `path.repo` gets registered, and
snapshots go somewhere durable.

**The vzdump exclusion is the non-obvious consequence, so it is called out explicitly.** The
nightly `nas-vmbackups` job selects VMs with "all except", which means inquisitor is enrolled
automatically the moment it exists (ADR 0009). Without clearing the per-disk Backup checkbox
on the 200 GiB data disk, that job would haul 200 GiB of index data across to the NAS every
single night — backing up a backup-grade volume of expendable derived data, over the network,
nightly. The checkbox is per-disk and defaults to enabled; forgetting it is silent, and shows
up as a backup window that grows from minutes to hours.
