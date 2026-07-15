# 0010 — Offsite replication: pull-based ZFS syncoid over Tailscale (refines 0005)
**Status:** Accepted — implementation pending (Phase 3)
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
