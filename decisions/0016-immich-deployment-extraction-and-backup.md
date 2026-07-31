# 0016 — Deployment model, extraction path, and backup consistency
**Status:** Accepted (Phase 5, 2026-07-24)
**Context:** Three coupled choices had to be settled before a single photo moved: how Immich
is installed, how the iCloud content gets out of Apple, and what a restore of the result
actually requires.
**Decision (deployment):** Docker Compose with the official images on a dedicated VM —
Immich's only supported installation method. The database image (Postgres 14 + pgvector +
VectorChord) is not substitutable with stock Postgres: face clustering runs as vector-index
queries inside the database. A new VM rather than shipyard because ML backfill would contend
with the Minecraft server, because vzdump restore stays independent, and because the blast
radius stays separate. Version pinned `IMMICH_VERSION=v3.0.3` rather than `release`, so
upgrades are deliberate.
**Decision (extraction):** The Apple takeout from privacy.apple.com, imported with
`immich-go from-icloud`, which reads dates and album membership from the takeout CSVs.
Recorded fact: requested 2026-07-24, delivered as 145.35 GB in 4 files, available until
2026-08-11. The ~17 GB gap against iCloud's reported 162.6 GB is expected — the takeout
excludes non-owned shared-album content, Recently Deleted, and Apple's derivatives; the
owned-originals set is what migrates.
**Decision (backup consistency):** vzdump is crash-consistent, and upstream warns against
copying a live `DB_DATA_LOCATION`, so database authority is Immich's built-in logical dump
(transactionally consistent) — daily 02:00, keep 14, landing in `UPLOAD_LOCATION/backups`.
That puts it on holocron, off the VM, inside the 03:00 snapshot, and inside the future
ADR 0010 replication set automatically. Three layers: vzdump (fast VM rebuild, auto-enrolled
in the nightly nas-vmbackups job) / the logical dump (database authority) / ZFS snapshots
(assets and dumps, point-in-time). A dump alone is not a restore — a restore is the dump
plus `UPLOAD_LOCATION`, always.
**Alternatives:** `icloudpd` (rejected — a third-party tool authenticating with the Apple ID
and 2FA; the takeout is mandatory anyway as the reconciliation manifest, immich-go consumes
it natively, and the credential never touches third-party code, so the control is eliminated
structurally rather than mitigated); stock Postgres (rejected — no vector index, no face
clustering); Immich on shipyard (rejected — CPU contention with Minecraft and a shared blast
radius); tracking `release` (rejected — unattended major upgrades on the lab's
highest-patch-cadence service); vzdump alone as the database backup (rejected —
crash-consistent is not transactionally consistent).
**Consequences:** Accepted cost on extraction: a 3–7 day wait, run in parallel with the
build, and Favorites status uncertain until the pilot import checks it. Known gap: the dumps
are unmonitored — interim mitigation is a monthly manual check, and the Phase 6 item is a
Wazuh rule on no-new-dump-in-48h. Cancellation gate: iCloud+ is not cancelled until the Pi 5
offsite appliance (ADR 0010) is verifiably replicating `holocron/photos`; until then the
takeout zips on falcon are the interim third copy and are retained.
