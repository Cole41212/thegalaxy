# 0010 — Offsite replication: pull-based ZFS syncoid over Tailscale (refines 0005)
**Status:** Implemented 2026-08-14 (Phase 5) — decided Phase 3. See the amendment below.
**Context:** 0005 committed to a Tailscale-only offsite Pi but left the mechanism open.
The offsite copy exists to survive ransomware, so the lab must not hold a credential that
can reach — let alone destroy — it.
**Decision:** Pull-based ZFS replication via syncoid over Tailscale. The Pi 5 runs Ubuntu
24.04 with a natively encrypted single-disk ZFS pool; the Pi initiates every transfer and
holds the only credential (a delegated zfs-allow user on archives), so a compromised lab
cannot reach the offsite copy.
**Alternatives:** restic to an append-only rest-server (kept as fallback if ZFS-on-USB
misbehaves); push-based designs (rejected — a lab-side credential can destroy offsite
history); cloud (cost, per 0005).
**Consequences:** Offsite copy is ransomware-resistant and encrypted at rest. Requires
TrueNAS snapshot tasks on the source datasets; the media subset needs a dedicated dataset
(e.g. holocron/media-keep); configs need a lab-side landing dataset (holocron/configs) to
ride along. Implementation deferred pending hardware (drive enclosure).

---

## Amendment — 2026-08-14 (implementation)

Implemented as **echo-base** (Raspberry Pi 5, Ubuntu Server 24.04 arm64) at mom's house in
NJ, where family remains resident. The decision and its security model stand as written and
were confirmed in practice: the Pi holds the only credential, that credential is read-only,
and no lab-side credential can reach the offsite copy. Four points amend the Consequences
above.

**1 — Single-disk, by choice rather than by constraint.** The original text assumed a
single disk without saying why. Available drives were 6 TB / 4 TB / 3 TB / 1 TB, and both
RAIDZ and mirror vdevs use only the smallest member's capacity, so mixed sizes make
redundant layouts absurd (6+4+3+1 as RAIDZ1 yields ~3 TB usable and strands ~10 TB). Two
mismatched mirrors (~4 TB usable) were rejected as a brutal capacity cost for a *tertiary*
copy whose source is already RAIDZ2; four separate top-level vdevs (~14 TB, zero redundancy)
were rejected because that quadruples failure probability to buy capacity the tier does not
need. Chosen: the single 6 TB disk, with the others retained as cold spares. The offsite
tier's job is geographic separation and ransomware isolation, not surviving its own drive
failure — holocron's RAIDZ2 is the resilience layer. Checksumming and verified send/recv are
retained; only self-healing is given up, and a detected error is answered by re-replicating
from an intact source. A second drive was deliberately **not** purchased: it would harden
the least critical of the three copies.

**2 — `holocron/media-keep` curation is superseded.** The Consequences above call for a
curated media subset on its own dataset. Unnecessary: `holocron/media` is 12.1 GB, small
enough to replicate whole. The offsite set is `holocron/photos` (149 GB) and
`holocron/media` (12.1 GB). Explicitly excluded: `holocron/vm-backups` (118 GB — vzdump
output, already a backup; replicating backups-of-backups inverts the value of a
capacity-limited tier), `holocron/files` (empty — Nextcloud deferred per ADR 0014),
`holocron/crafty-backups`, and `holocron/pibackup`.

**3 — `--no-sync-snap`, and the destroy verb that was not granted.** syncoid by default
creates its own sync snapshot on the source and destroys the previous one, which failed —
correctly — with `cannot destroy snapshots: permission denied`, because the delegated
`pibackup` user has no destroy verb. Three options were weighed: leave it (syncoid snapshots
accumulate on holocron forever); grant `destroy` (rejected — that hands a delete verb to a
credential living in another house, eroding precisely the property this ADR exists to
create); or `--no-sync-snap` (chosen — syncoid replicates from the existing TrueNAS periodic
snapshots and creates nothing that needs cleaning up). Structural elimination over privilege
escalation, the same reasoning that dropped the MGMT tailnet route in Phase 3 rather than
ACL-ing it. Consequence: replication captures the most recent periodic snapshot rather than
live state, which is the intended behaviour for a nightly backup. The delegated set is
`send,snapshot,hold,bookmark,mount`; `receive`, `create`, `destroy`, and `rollback` are
deliberately absent.

**4 — The USB-attached-ZFS falsifier did not fire.** The ADR kept restic-to-an-append-only-
rest-server as the fallback if ZFS-on-USB misbehaved. Across a 154 GB sustained seed write,
`dmesg` showed zero USB resets, zero re-enumeration, and zero SCSI errors, with the
enclosure negotiating UAS rather than falling back to BOT. The fallback stays on the shelf
unused.

**Still open — configs are not in the replication set.** `holocron/configs` was never
created, so the OPNsense XML export and the switch config remain only in the local,
gitignored `config-backups/` folder and have no offsite copy. The Consequences above listed
this as a requirement and it is unmet.
TODO(cole): decide whether to open this as a tracked follow-up or accept it explicitly.
