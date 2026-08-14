# Runbook — Immich Restore (vault / VM 107)

Backup design and rationale: [ADR 0016](../decisions/0016-immich-deployment-extraction-and-backup.md)
and [ADR 0015](../decisions/0015-immich-data-placement-and-rw-nfs.md).
Build log: [phases/phase-5-vault-immich.md](../phases/phase-5-vault-immich.md).
Offsite procedures: [runbooks/offsite-replication.md](offsite-replication.md).

> **A database dump alone is not a restore.** A restore requires BOTH the database dump AND
> the contents of `UPLOAD_LOCATION`. Neither is a backup by itself. Immich stores asset file
> paths in the database and never rescans the library to rebuild them — restoring only the
> files gives you an unindexed pile with no albums, people, or edits; restoring only the
> database gives you an index pointing at nothing.

Read that before touching anything below. Every scenario in this runbook is an application of
it.

## What exists and where

Four independent artifacts. Only the first two are the restore; the other two are speed and
point-in-time granularity.

| Artifact | Location | Produced by | Retention |
|---|---|---|---|
| **Logical DB dump** — database authority | `/mnt/photos/backups` on vault = `holocron/photos/backups` on archives | Immich's built-in backup scheduler, daily 02:00 | keep 14 |
| **Asset library** — `UPLOAD_LOCATION` | `/mnt/photos` on vault = `holocron/photos` on archives (NFS `rw,hard`) | Immich, continuously | live data |
| vzdump of VM 107 | `nas-vmbackups` (TrueNAS `vm-backups` dataset) | nightly Proxmox job, auto-enrolled by "all except 100/102" | per job policy |
| ZFS snapshots of `holocron/photos` | archives, `.zfs/snapshot/` | TrueNAS periodic task, 03:00 daily | keep 2 weeks |
| Offsite replica | echo-base `carbonite/photos` | syncoid pull, 04:30 daily | mirrors source snapshots |

Points that decide which path you take:

- **The dump lives inside `UPLOAD_LOCATION`, deliberately.** That single placement puts it off
  the VM, inside the 03:00 snapshot, and inside the offsite replication set with no separate
  job to maintain. It also means `carbonite/photos` carries *both halves of a restore in one
  dataset* — the reason offsite is a single replication stream.
- **The 03:00 snapshot contains that day's 02:00 dump.** The ordering is not incidental: any
  snapshot you roll back to holds a dump that is at most one hour older than the assets beside
  it.
- **vzdump is NOT the database's authority.** It is crash-consistent, and upstream warns
  against copying a live `DB_DATA_LOCATION`. Treat it as a fast VM rebuild path — OS, Docker,
  the compose files, `.env`, and a Postgres directory you should expect to discard. Restoring
  VM 107 from vzdump and calling it done leaves you with a database that was copied mid-write.
- **`DB_DATA_LOCATION` is `/opt/immich/postgres`, on vault's local disk** (ADR 0015 — Postgres
  needs POSIX locking NFS does not reliably provide). It is not on holocron and it is not in
  the snapshot or the offsite set. Nothing of value is lost there, because the logical dump is
  the authority.

## Before you restore anything: verify the dump

```bash
ls -lt /mnt/photos/backups | head
```

Newest first. Confirm the timestamp is what you expect and the size is plausible — a dump that
is kilobytes when yesterday's was hundreds of megabytes is a failed dump that still wrote a
file.

```bash
gunzip -t /mnt/photos/backups/immich-db-backup-<timestamp>-v<ver>-pg<ver>.sql.gz
```

Silence means the gzip stream is intact. **An unverified dump is a hope, not a backup.** Do
this *before* you tear down a working stack, not after — the point of testing first is that a
failed test still leaves you the option of the next-newest dump, or of a snapshot, while the
current state is still intact.

If the newest dump fails, walk backwards through the retained 14, or pull one out of a ZFS
snapshot (see scenario (a), step 2).

## Version matching — read this before redeploying

**A dump must be restored into the same major Immich version it came from.** Immich applies
schema migrations on startup; a dump from an older schema loaded into a newer server, or the
reverse, is not a supported path and will fail in ways that are expensive to unpick.

The dump filename encodes the versions it was taken under:

```
immich-db-backup-YYYYMMDDTHHMMSS-v3.0.3-pg14.19.sql.gz
                                  ^^^^^^ ^^^^^^^
                                  Immich  Postgres
```

**Read the version out of the filename and pin `IMMICH_VERSION` in `/opt/immich/.env` to
match, before `docker compose up`.** This is the whole reason the deployment pins a version
rather than tracking `release` (ADR 0016) — a `release` tag would silently pull a newer major
during exactly the rebuild where you need the old one.

Current deployment: **`v3.0.3`**, database image
`14-vectorchord0.4.3-pgvector0.2.0`. That database image is not substitutable with stock
Postgres — face clustering runs as vector-index queries inside the database, so a restore into
plain `postgres:14` loses the extensions the dump expects.

Once the restore is verified healthy, upgrading is a separate, deliberate act. Do not combine
it with a restore.

---

## Scenario (a) — Database corrupted, VM and files intact

The common case: Immich errors on startup or the timeline is wrong, but vault is up and
`/mnt/photos` is mounted and correct.

**Do not touch `/mnt/photos`.** The assets are fine. This scenario restores the index over an
intact library.

1. **Verify the dump you intend to use** (section above). Do this first.

2. If the newest dumps are also bad, retrieve an older one from a ZFS snapshot on archives.
   Snapshot directories automount on access:
   ```bash
   ls /mnt/holocron/photos/.zfs/snapshot/
   ```
   then copy the dump out of the chosen snapshot:
   ```bash
   cp /mnt/holocron/photos/.zfs/snapshot/<snap>/backups/<dump>.sql.gz /mnt/holocron/photos/backups/
   ```
   Copy it out rather than reading it in place, so the restore does not depend on a snapshot
   staying mounted. Note the trade: a dump from an older snapshot indexes assets that exist,
   but will not know about assets added after it was taken (see "After any restore" below).

3. **Confirm the version** from the filename and check `/opt/immich/.env` matches.

4. Stop the stack:
   ```bash
   cd /opt/immich && docker compose down -v
   ```
   `-v` removes the named `model-cache` volume — it is a regenerable ML cache, not data.
   It does **not** remove `DB_DATA_LOCATION`, which is a bind mount to `/opt/immich/postgres`
   and has to be cleared by hand in the next step.

5. Clear the corrupted database directory:
   ```bash
   sudo rm -rf /opt/immich/postgres
   ```
   Verify the path before running this. It is local-only and regenerable from the dump; the
   irreplaceable data is on NFS and untouched.

6. Bring up only Postgres, so nothing writes to the database while it is being loaded:
   ```bash
   docker compose create
   docker start immich_postgres
   ```
   Give it a few seconds to initialize and check it is accepting connections:
   ```bash
   docker exec immich_postgres pg_isready
   ```

7. Load the dump. `DB_USERNAME` and `DB_DATABASE_NAME` come from `/opt/immich/.env`:
   ```bash
   gunzip --stdout /mnt/photos/backups/<dump>.sql.gz \
     | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" \
     | docker exec -i immich_postgres psql --dbname=postgres --username=<DB_USERNAME>
   ```
   The `sed` is upstream's documented workaround: the dump sets an empty `search_path`, which
   breaks restoring the vector extension objects. Do not drop it. **Check the restore section
   of the docs for the pinned version before running this** — the exact command is
   version-specific, and the version you are restoring into is the one whose docs apply.

8. Bring the full stack up:
   ```bash
   docker compose up -d
   ```

9. Verify (section below).

## Scenario (b) — VM 107 lost, holocron intact

vault is gone — failed disk, botched upgrade, deleted VM. The library and the dumps are safe
on the NAS, which means this is a rebuild of a stateless consumer, not a data recovery.

Two ways in. **Restoring the vzdump is faster; a clean build is more predictable.** Either
way, the database comes from the dump, never from whatever Postgres directory the vzdump
carried.

1. **Rebuild the machine.**
   - *Fast path:* restore VM 107 from `nas-vmbackups` in the Proxmox GUI. Expect the restored
     `/opt/immich/postgres` to be crash-consistent garbage — you will delete it in step 4
     regardless.
   - *Clean path:* build a new VM to the Phase 5 spec in
     [phases/phase-5-vault-immich.md](../phases/phase-5-vault-immich.md) — VM 107, Ubuntu
     Server 24.04 **standard** install, q35 + OVMF (uncheck Pre-Enroll Keys), 3 vCPU host type
     with CPU units 50, 12 GB with ballooning off, 80 GB scsi0 on the NVMe with discard + SSD
     emulation + iothread, vmbr1 tag 30, qemu-guest-agent. Then reapply the OS-level
     configuration from that document: `Acquire::ForceIPv4`, the local `immich` account at
     UID/GID 3001 (nologin), and the `/etc/hosts` self-hostname line pointing at 10.0.30.40
     rather than 127.0.1.1.

2. **Restore the address, not just the VM.** vault is DHCP in-OS with a Kea reservation
   pinning 10.0.30.40. A rebuilt VM has a new MAC, so **update the Kea reservation to the new
   MAC** or the reservation silently stops applying. Confirm before continuing:
   ```bash
   ip -4 addr show
   getent hosts vault
   ```
   Both must show 10.0.30.40. The NFS export on archives is scoped to `10.0.30.40/32` — on any
   other address the mount is refused, and that refusal is the export working correctly.

3. **Re-mount NFS** per ADR 0015 — `10.0.30.20:/mnt/holocron/photos` → `/mnt/photos`,
   `nfs4 rw,hard,nconnect=4,noatime,_netdev,x-systemd.automount`. Never `soft` on this mount:
   a soft mount that times out on a WRITE returns EIO without the client knowing whether the
   bytes landed, which is silent divergence between the database and the library.

   Verify as the authorized identity, not as root:
   ```bash
   findmnt -t nfs4 /mnt/photos
   sudo -u immich touch /mnt/photos/.permcheck && sudo -u immich rm /mnt/photos/.permcheck
   ```
   `sudo ls /mnt/photos` returning `Permission denied` is **root squash working as designed**,
   not a mount failure. Reading that as a mount problem sent the 2026-08-05 investigation down
   the wrong layer entirely.

4. **Re-add the Docker → automount ordering dependency.** A systemd override on
   `docker.service` with `RequiresMountsFor=/mnt/photos`. Without it, Docker can start before
   the automount has fired, `immich_server` finds no bind-mount source, and it exits 128 while
   the other three containers come up healthy — a stack that looks three-quarters fine and is
   not running. This is a rebuild step, not a troubleshooting step.

5. **Redeploy the stack pinned to the version the dump came from.** Read the version out of
   the dump filename first (see above), set `IMMICH_VERSION` in `/opt/immich/.env` to match,
   restore `UPLOAD_LOCATION=/mnt/photos` and `DB_DATA_LOCATION=/opt/immich/postgres`, keep
   `user: "3001:3001"` on `immich-server`, and leave the `hwaccel` blocks commented out.

   `DB_PASSWORD` exists only in `/opt/immich/.env` and is deliberately not in this repo. On the
   clean path you are setting a new one — it just has to be internally consistent across the
   compose file and the fresh database, since the dump carries no password.

6. **Restore the dump** — scenario (a), steps 4 through 8, starting from a stack that is down.

7. Verify (below), then re-check the admin settings that are not carried in the dump's scope
   if anything looks off — in particular the transcode policy ("Don't transcode any videos",
   target resolution Original, threads 2) and the 02:00 backup schedule.

## Scenario (c) — Total loss of the primary site

executor, archives, and holocron are all gone. What survives is `carbonite` on echo-base, at
another site, on the tailnet.

**`carbonite/photos` carries both halves of the restore** — all six Immich directories
including `backups`, so the assets and the logical dumps arrive together. That is the design
property this scenario exists to cash in.

Two structural facts to hold onto before starting:

- **The read-only `pibackup` delegation does not obstruct this.** It constrains the *scheduled
  pull*, not you. A restore runs as root at both ends against a new destination pool, so
  nothing here needs the credential that was deliberately kept powerless.
- **The pool key lives on echo-base at `/root/.carbonite.key`** and never left it. That is
  what makes a stolen Pi yield ciphertext — and it also means the surviving Pi must be intact
  and unlockable for this scenario to work at all. Confirm before planning around it:
  ```bash
  zfs get keystatus carbonite/photos
  ```
  Expect `available`.

1. **Rebuild the foundation** per the restore chain in
   [docs/backup-strategy.md](../docs/backup-strategy.md): Proxmox on new hardware, then tarkin
   and archives, then a new `holocron`. Routing and the NAS have to exist before there is
   anywhere to send photos.

2. **Move the data back.** Direction reverses — echo-base is now the source. Run from
   echo-base, sending into the rebuilt archives:
   ```bash
   sudo syncoid --sshkey=/root/.ssh/id_ed25519 carbonite/photos <user>@<new-archives>:holocron/photos
   ```
   The account on the *new* archives needs `receive` and `create`, which `pibackup` does not
   have and should not be given. Use an admin account for the restore and rebuild the
   least-privilege `pibackup` delegation afterwards, when the normal pull direction resumes.

   For a partial or urgent recovery, files can be read directly out of a snapshot instead —
   `sudo -i` first, then `/carbonite/photos/.zfs/snapshot/<snap>/...`. `sudo cd` does not work;
   `cd` is a shell builtin.

3. **Rebuild vault** — scenario (b), steps 1 through 5.

4. **Restore the dump** from the replicated `backups/` directory, now landed under
   `/mnt/photos/backups`. Verify with `gunzip -t` first; the replica is checksummed but the
   dump inside it is only ever as good as the night it was written.

5. Re-establish the offsite tier last: recreate `pibackup` with the
   `send,snapshot,hold,bookmark,mount` delegation and re-point the syncoid timers. Until that
   is done you are running on one copy.

---

## After any restore — verify

```bash
cd /opt/immich && docker compose ps
```

Expect four healthy containers: `immich_server`, `immich_machine_learning`, `immich_postgres`,
`immich_redis`.

```bash
docker compose logs immich_server | grep -i -e error -e eacces
```

**No EACCES on `/data` paths.** EACCES means the identity chain broke — process UID, NFS wire
UID, and dataset owner must all be 3001. Check `user: "3001:3001"` in the compose file and
`ls -ld /mnt/photos` showing `drwxrwx--- 3001 3001`.

Then, in the web UI at `http://10.0.30.40:2283`:

- **Asset count** matches what you expect (reference point: 17,436 assets — 13,953 images +
  3,483 videos, 148.9 GiB).
- **Thumbnails render and originals open.** A thumbnail loading proves the database row; the
  original opening proves the row still points at a real file. Check both — that pairing is
  the whole "dump plus `UPLOAD_LOCATION`" claim, tested.
- **Albums, people, and favorites are present.** These live only in the database and are the
  fastest signal that the dump loaded rather than the schema merely being created.
- **A new upload from the mobile app succeeds.** Proves the write path, not just the read path.
- **The next 02:00 dump appears** in `/mnt/photos/backups`. A restored instance that is not
  producing backups is not restored, it is running.

Do not run a ML re-index to "fix" a restore. Face clustering and smart search results are in
the database and come back with the dump; re-running them is hours of CPU on the shared 7700T
for no gain.

### If the dump predates the assets

A dump restored from a snapshot or from offsite may be older than the newest files in
`UPLOAD_LOCATION`. The result is assets present on disk that the database does not know about
— they will not appear in the timeline. Immich does not rescan `UPLOAD_LOCATION` to find them.
The recovery is to re-upload the affected period from the original device, which the mobile
app's own backup will do for anything still on the phone. Expect this, rather than discovering
it as a mystery gap weeks later.

## Known gap — the dumps are unmonitored

**Nothing alerts if the dumps stop.** The scheduler runs inside Immich, and a failed or
silently-skipped backup produces no notification anywhere. This is a documented consequence in
ADR 0016, not an oversight — and it has already happened once in a related shape: the
2026-08-05 outage was diagnosed partly *because* the newest dump was seven days stale, which
means the staleness existed for a week with nothing raising a hand.

**Interim mitigation — monthly manual check:**

```bash
ls -lt /mnt/photos/backups | head -3
```

The newest file must be recent (within a day) and non-trivially sized. A dump file that exists
but is a few kilobytes is a failed dump, and a size check catches it where a presence check
does not.

**Phase 6 item:** a Wazuh rule alerting on no-new-dump-in-48h. This is the same dead-man's
switch shape as the healthchecks.io monitor on offsite replication — absence of a success
signal is the alert — applied to the one backup layer that currently has no such monitor.
