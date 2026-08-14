# Phase 5 — vault (Immich) & the iCloud Migration

**Goal:** Replace a paid iCloud+ subscription with a self-hosted photo platform: build vault
(VM 107), run Immich against a dedicated ZFS dataset over read-write NFS, extract ~145 GB of
owned originals from Apple, and cut the phone over — without ever holding fewer than three
copies of the library. Along the way: the platform choice (ADR 0014), the data-placement and
rw-NFS semantics (ADR 0015), the deployment/extraction/backup model (ADR 0016), and the
access and TLS posture (ADR 0017).

**Status:** 🚧 In progress — migration complete and the ADR 0016 cancellation gate satisfied
(offsite replication verified by restore test, 2026-08-14); the two-week soak, the iCloud+
cancellation itself, and post-phase cleanup remain

---

## Decisions & Rationale

### Photo platform: Immich, not Nextcloud (ADR 0014)

Phase 5 was scoped as Nextcloud since Phase 2, but the requirement that actually exists is
replacing iCloud+ — ~145 GB of owned originals, target ~17k assets — with mobile auto-backup,
a timeline, albums, face recognition, semantic search, and text-in-image (OCR). Nextcloud
reaches most of that only through add-ons (Memories, Recognize) bolted onto a file-sync core,
with weaker mobile backup; Immich is purpose-built for exactly this shape of workload.
PhotoPrism lost on mobile-client and background-backup maturity, and running both Nextcloud
and Immich was rejected outright — two applications owning the same photos is a data-
ownership problem, not a feature. General file sync/share is deferred to a Phase 6 planning
revisit, with the triggers written down in
[ADR 0014](../decisions/0014-photo-platform-immich.md): a need for external share links,
Phase 6 wanting an auth-log-rich application target, or SMB proving insufficient for remote
file access. The consequence to keep in view is patch
cadence — Immich does not backport fixes, so this box will move faster than the rest of the
lab.

### Data placement and read-write NFS semantics (ADR 0015)

Immich has two storage classes with opposite requirements, and treating them alike is the
way to corrupt the library. Postgres needs POSIX file locking that NFS does not reliably
provide — upstream explicitly unsupports network shares for `DB_DATA_LOCATION` — so the
database lives on vault's local NVMe-backed disk. The asset library is bulk data, which the
Phase 2 rule puts on holocron; it also has to stay on one filesystem, because
`UPLOAD_LOCATION` is six subfolders Immich moves files between with `rename(2)`, which fails
EXDEV across a filesystem boundary. Hence a dedicated dataset `holocron/photos`
(recordsize=1M for large media, atime=off) with the whole tree beneath one mount.

The mount is `rw,hard` — deliberately the opposite of cantina's `ro,soft`. A soft mount that
times out on a WRITE returns EIO without the client knowing whether the bytes landed, which
is silent divergence between Immich's database and its library. Cantina's rationale (a failed
read is a failed playback; nothing is lost) simply does not transfer to a write path.
`x-systemd.automount` rather than `nofail` for the same reason: nofail plus a down NAS means
Docker starts and Immich writes into an empty local mountpoint, silently splitting the
library, whereas automount makes the mountpoint a kernel trigger that blocks instead.

The ADR 0009 deadlock analysis does *not* apply here, and that is worth being explicit about
rather than assuming: vault↔archives is same-subnet on vmbr1 and never transits tarkin, and
vault sits on no path archives depends on, so a stalled mount hangs Immich and nothing else.
Falsifier recorded in [ADR 0015](../decisions/0015-immich-data-placement-and-rw-nfs.md): if a
vault NFS stall is ever observed to affect anything outside vault, the premise is wrong and
the design reopens.

Identity is one number end to end — TrueNAS user and group `immich` at UID/GID 3001, dataset
0770 `immich:immich`, the NFS export scoped to 10.0.30.40/32 with maproot left empty so root
squash stays on (root on vault arrives as `nobody` and cannot touch the dataset), and the
containers running `user: "3001:3001"`. Process UID = NFS wire UID = dataset owner. The
documented fallback lever, if EACCES ever appears on `/data` paths, is maproot
`immich`/`immich` plus containers as root — writes still land as 3001, but container-escape
protection is lost. **Lever used: none** — the UID match works with root squash on, verified
below.

### Deployment, extraction, and backup consistency (ADR 0016)

Docker Compose with the official images is Immich's only supported installation method, and
the database image (Postgres 14 + pgvector + VectorChord) is not substitutable with stock
Postgres — face clustering runs as vector-index queries inside the database. A dedicated VM
rather than reusing shipyard, because ML backfill would contend with the Minecraft server,
vzdump restore stays independent, and the blast radius stays separate. `IMMICH_VERSION` is
pinned to `v3.0.3` rather than tracking `release`, so upgrades on the lab's fastest-moving
service are deliberate acts.

Extraction goes through Apple's own takeout at privacy.apple.com, imported with `immich-go
from-icloud`, rather than `icloudpd`. icloudpd is a third-party tool that authenticates with
the Apple ID and 2FA; the takeout is mandatory anyway as the reconciliation manifest,
immich-go consumes its CSVs natively for dates and album membership, and the credential never
touches third-party code — the control is eliminated structurally rather than mitigated. Cost
accepted: a 3–7 day wait (run in parallel with the build) and Favorites status uncertain
until the pilot import checks it.

Backups are three layers, because vzdump is crash-consistent and upstream warns against
copying a live `DB_DATA_LOCATION`. Database authority is Immich's built-in logical dump
(transactionally consistent), daily 02:00 keep 14, landing in `UPLOAD_LOCATION/backups` —
which puts it on holocron, off the VM, inside the 03:00 snapshot, and inside the future
ADR 0010 replication set automatically. vzdump gives a fast VM rebuild; ZFS snapshots give
point-in-time for assets and dumps. The rule to remember at restore time is that a dump alone
is not a restore: it is the dump *plus* `UPLOAD_LOCATION`, always. Cancellation gate, per
[ADR 0016](../decisions/0016-immich-deployment-extraction-and-backup.md): iCloud+ is not
cancelled until the Pi 5 offsite appliance is verifiably replicating `holocron/photos`; until
then the takeout zips on falcon are the interim third copy.

### Access and TLS: plain HTTP, time-boxed (ADR 0017)

There is no domain yet — Phase 8 acquires one — and Immich's FAQ is explicit that self-signed
certificates break mobile video playback and upload. That leaves a VPN or a real certificate,
and the lab already has the VPN: remote access rides the existing phantom subnet route, so
vault gets no tailnet node of its own and ADR 0011's single-ingress property stays intact.
One URL for LAN and tailnet alike (`http://10.0.30.40:2283`) because URL-switching has
multiple open upstream bugs. No new firewall rules were needed — single user on TRUSTED, no
FAMILY pinhole. The accepted risk is cleartext on VLAN 30 and TRUSTED WiFi, time-boxed
against a Phase 8 remediation (Caddy + ACME DNS-01 on the portfolio domain, zero inbound
ports) and recorded in [docs/threat-model.md](../docs/threat-model.md).

[ADR 0017](../decisions/0017-vault-access-and-tls-model.md) also settles three *outbound*
egress questions, which are consent decisions distinct from the no-inbound-exposure posture:
Version Check (version.immich.cloud) stays **on** — it sends only the running version string
and returns update-available notices, which is exactly what serves ADR 0014's patch-cadence
consequence; map tiles (tiles.immich.cloud) stay **on** — tiles load on demand when the map
view opens, so the CDN can infer approximate photo geography from the regions requested, but
no photos and no exact coordinates leave the box and place names come from the local
reverse-geocoder; Google Cast is **off** — it pulls external Google code into the web client
for no stated need, the primary client being the iOS app.

---

## Checklist

| Step | Status | Notes |
|------|--------|-------|
| Apple takeout requested | ✅ | 2026-07-24 — delivered 145.35 GB in 4 files, available until 2026-08-11 |
| ADRs 0014–0017 recorded | ✅ | Platform · data placement/rw NFS · deployment/extraction/backup · access & TLS |
| `holocron/photos` dataset + `immich` identity | ✅ | recordsize=1M, atime=off; TrueNAS user+group `immich` 3001; NFS export scoped 10.0.30.40/32, maproot empty |
| Snapshot task on `holocron/photos` | ✅ | 03:00 daily, keep 2 weeks, empty snapshots allowed |
| vault built (VM 107) | ✅ | Ubuntu 24.04 standard install, q35/OVMF, 3 vCPU / 12 GB (ballooning off) / 80 GB NVMe, CPU units 50, vmbr1 tag 30, guest agent, discard+ssd+iothread on scsi0 |
| DHCP + Kea reservation + DNS | ✅ | 10.0.30.40; FQDN resolves (cantina's pattern, not static-in-OS) |
| NFS mounted `rw,hard` | ✅ | nconnect=4, vers=4.2; write proven as UID 3001; root correctly denied by root-squash |
| Local `immich` account on vault | ✅ | UID/GID 3001, nologin — mirrors the NAS account |
| `/etc/hosts` self-hostname corrected | ✅ | 127.0.1.1 → 10.0.30.40 (log entry below) |
| Docker CE + Immich v3.0.3 stack | ✅ | Official Docker repo; 4 containers healthy, EACCES-clean |
| Admin settings applied | ✅ | Storage template enabled pre-import, transcode policy, job concurrency, DB backup schedule, egress toggles |
| Takeout extracted on falcon | ✅ | 145.35 GB across 4 parts → staging |
| Pilot import passed | ✅ | ~10 assets via `upload from-folder --into-album=pilot` — dates, RAW, slo-mo, Live Photo pairing |
| holocron drive `sdd` replaced | ✅ | 2026-08-11 — media failure; resilvered 0 errors, pool back to ONLINE (log below) |
| Bulk import | ✅ | `immich-go upload from-icloud` — 17,436 assets discovered, 148.9 GiB stored; succeeded on the third run |
| Albums reconstructed | ✅ | From the takeout CSVs — untested by the pilot, which used `from-folder`, so validated here |
| ML processing complete | ✅ | Smart Search and face detection done; OCR finishing in the background |
| Reconciliation vs source | ✅ | 13,953 images + 3,483 videos against the Photos app's 14,325 + 3,119; 8 checksum duplicates discarded |
| Screenshots archived | ✅ | Out of the main timeline — also resolves most of the wrong-date population (follow-up below) |
| LTE verification | ⬜ | Immich reachable over the tailnet off home WiFi |
| DB dump verified | ⬜ | First 02:00 dump present in `UPLOAD_LOCATION/backups` |
| vzdump enrolled | ⬜ | Auto-enrolled via the job's "all except" selection (ADR 0009) — confirm |
| Snapshot fired | ⬜ | First 03:00 snapshot on `holocron/photos` |
| Phone cutover (iCloud Photos off) | ✅ | 2026-08-13 — disabled with "Remove from iPhone"; device library ~145 GB → ~12 GB |
| Immich mobile app configured | ✅ | Camera-roll permission granted explicitly; backup enabled on all device photos |
| New-photo round trip verified | ✅ | Photo taken on the device uploaded to the server immediately |
| Two clean weeks | ⬜ | Auto-backup running, no gaps |
| Pi 5 replicating `holocron/photos` | ✅ | 2026-08-14 — echo-base seeded 154 GB; **restore test passed** (sha256 match against source). ADR 0010 implemented, ADR 0016 gate satisfied |
| iCloud+ cancelled | TODO(cole) | The gate above is now met, so this is unblocked. Renewal is 2026-09-09 — confirm whether to cancel now or ride to renewal |
| Cleanup | ⬜ | Revoke the API key, restore falcon's power settings, delete the takeout (no longer needed as the interim third copy now that offsite is live) |

---

## vault (VM 107) build config

| Setting | Value |
|---------|-------|
| VM ID | 107 |
| Hostname | vault |
| OS | Ubuntu Server 24.04 LTS — **standard** install (not minimized; cantina's Phase 4 lesson) |
| Machine | q35 |
| BIOS | OVMF (UEFI) + EFI disk |
| CPU | 3 vCPU, host type; CPU units 50 |
| RAM | 12288 MB, ballooning OFF |
| Disk | 80 GB `scsi0` on the NVMe store — discard + SSD emulation + iothread; VirtIO SCSI single |
| Network | vmbr1, VLAN tag 30, VirtIO |
| Guest agent | qemu-guest-agent enabled |
| IP | 10.0.30.40 — DHCP in-OS + Kea reservation (network/ip-scheme.md) |

CPU units 50 (default 100) deliberately deprioritizes vault against its neighbours on the
shared 7700T: an ML backfill is a long, greedy, entirely deferrable workload, and tarkin's
routing is not.

### OS-level configuration

- **APT:** `Acquire::ForceIPv4 "true"` — the lab is IPv4-only; same finding as cantina
  (Phase 4 log #5), applied at build time instead of after burning an hour.
- **Local service account:** `immich` uid=3001 gid=3001, nologin, no home — mirrors the
  TrueNAS `immich` account and the container's `user: "3001:3001"`.
- **`/etc/hosts`:** the self-hostname line points at 10.0.30.40, not 127.0.1.1 (log below).
- **NFS:** `10.0.30.20:/mnt/holocron/photos` → `/mnt/photos`, mounted
  `nfs4 rw,hard,nconnect=4,noatime,_netdev,x-systemd.automount` (ADR 0015).
- **Docker:** CE from download.docker.com, keyring at `/etc/apt/keyrings/docker.asc`.

### Immich stack

- **Version:** pinned `IMMICH_VERSION=v3.0.3` — not `release`.
- **Paths:** `UPLOAD_LOCATION=/mnt/photos`, `DB_DATA_LOCATION=/opt/immich/postgres`.
- **Identity:** `user: "3001:3001"` on `immich-server` only; postgres, valkey, and
  machine-learning are left unchanged.
- **Hardware accel:** the `hwaccel` extends blocks stay commented out — CPU-only ML by
  design.
- **Secret:** `DB_PASSWORD` exists only in `/opt/immich/.env` on the VM. It is never written
  into this repo, and `config-backups/` is gitignored regardless.

### Immich admin settings

| Setting | Value | Why |
|---|---|---|
| Storage template | `{{y}}/{{y}}-{{MM}}/{{filename}}` — engine on, hash verification on | Enabled **before** import; turning it on afterwards renames the whole library over NFS |
| Transcode policy | **"Don't transcode any videos"** — changed 2026-08-13, was "higher than target res or not accepted" | The original policy was OR-joined and Target Resolution was still 720p, so every 4K video transcoded anyway (bulk-import log below). Permanent: the iOS app decodes 4K HEVC in hardware, and Immich v3 transcodes on the fly for browsers |
| Target resolution | Original — was 720p | The clause that defeated the accepted-codec list |
| Transcode threads | 2 — was 0 | Immich documents 0 as "maximizes utilization"; unlimited on a 3-vCPU VM starved upload and hashing |
| Job concurrency | Smart Search 2, Face Detection 2 | Caps ML so tarkin keeps CPU headroom on the shared 7700T |
| Hardware acceleration | Disabled | CPU-only ML by design (the RX 5700 belongs to cantina) |
| Database backup | 02:00 daily, keep 14 | The logical dump is database authority (ADR 0016) |
| External egress | Version Check on · Map on · Google Cast off | ADR 0017 |

---

## Verification evidence

- **Mount, from `findmnt`:** `/mnt/photos` is `nfs4` with
  `rw,hard,nconnect=4,vers=4.2,rsize=1048576,wsize=1048576,sec=sys` — the ADR 0015 options
  as actually negotiated, not as written in fstab.
- **Permission proof:** `sudo -u immich touch /mnt/photos/.permcheck` then `rm` →
  **WRITE-OK / DELETE-OK**; `sudo ls /mnt/photos` as root → **Permission denied**. Root
  squash confirmed working in the direction that matters: the service identity writes, root
  cannot.
- **Ownership:** `ls -ld /mnt/photos` → `drwxrwx--- 3001 3001` — 0770, owned by the same
  numeric identity the containers run as.
- **DNS:** `dig @10.0.30.53 vault.galaxy.internal +short` → `10.0.30.40`;
  `getent hosts vault` → `10.0.30.40` (after the `/etc/hosts` fix); `dig flurry.com` →
  `0.0.0.0`, so vault resolves through Pi-hole and inherits the filtering.
- **Containers:** `docker compose ps` → `immich_server` (v3.0.3, 2283 the only published
  port), `immich_machine_learning` (v3.0.3, internal :3003), `immich_postgres`
  (`14-vectorchord0.4.3-pgvector0.2.0`), `immich_redis` (valkey) — all healthy.
- **Startup log:** feature flags `smartSearch` / `facialRecognition` / `ocr` / `sidecar`
  true, `importFaces` false; ML reachable over the internal service name; 227,792 geodata
  records imported (the local reverse-geocoder, per ADR 0017); **no EACCES** anywhere on the
  NFS paths — the identity chain holds inside the running container.

### Migration (2026-08-13)

- **Import:** 17,436 assets discovered — 13,953 images + 3,483 videos — reconciling closely
  against the Photos app's own 14,325 photos + 3,119 videos; 8 local duplicates discarded by
  checksum.
- **Storage:** 148.9 GiB in Immich against a 145.35 GB source takeout.
- **Albums:** reconstructed from the takeout CSVs via `immich-go upload from-icloud`.
- **Asset types:** Live Photos paired into motion assets; RAW (`.DNG`) and slo-mo verified.
- **Favorites:** transferred intact — closes the open cost ADR 0016 accepted on the takeout
  route ("Favorites status uncertain until the pilot import checks it").
- **ML:** face clustering and semantic search operational; OCR completing in the background.
- **Round trip:** a new photo taken on the device reached the server immediately.

### Offsite (2026-08-14)

- **Seed:** 154 GB transferred at ~76 MiB/s — the full snapshot history, not just current
  state.
- **Sizes:** `zfs list` → `carbonite/photos` USED 150G / REFER 149G against a 149G source;
  `carbonite/media` 12.1G. 5.17T available.
- **Snapshot history intact:** `auto-2026-07-31_03-00` through `auto-2026-08-13_03-00`
  present on photos — the offsite copy has the same point-in-time recovery granularity as
  the source, not a flattened single copy.
- **Encryption at rest:** `zfs get encryption,keystatus carbonite/photos` → `aes-256-gcm`,
  `available`.
- **Both halves of a restore present:** all six Immich directories replicated (`backups`,
  `encoded-video`, `library`, `profile`, `thumbs`, `upload`) — including `backups`, so per
  ADR 0016 the replica holds the logical DB dump *and* `UPLOAD_LOCATION`, not just assets.
- **RESTORE TEST PASSED:** `sha256sum` of `library/admin/2019/2019-11/IMG_0462.jpg` on
  carbonite matched the source on archives byte for byte. This is the ADR 0016 gate as
  written — not "a copy exists" but "data was proven recoverable."
- **USB stability:** `dmesg` across the 154 GB sustained write showed zero USB resets, zero
  re-enumeration, and zero SCSI errors, all entries dating from boot. The ADR 0010
  USB-attached-ZFS falsifier did not fire; the enclosure negotiated UAS rather than falling
  back to BOT.
- **Cold-reboot test:** pool auto-imported, key auto-loaded, dataset mounted — no manual
  intervention.

---

## Log

### 2026-07-24 — Phase started: takeout requested, ADRs drafted, storage + VM built

Phase 5 opened by requesting the Apple takeout first, because it is the long pole — 145.35 GB
delivered in 4 files, available until 2026-08-11 — and everything else was built in parallel
with the wait. ADRs 0014–0017 drafted: platform choice, data placement and rw-NFS semantics,
deployment/extraction/backup model, access and TLS.

On archives: dataset `holocron/photos` created with recordsize=1M and atime=off, the `immich`
user and group created at UID/GID 3001, the dataset set 0770 `immich:immich`, and an NFS
export scoped to 10.0.30.40/32 with maproot left empty so root squash stays on. Added a
Data Protection snapshot task on `holocron/photos` — 03:00 daily, keep 2 weeks, empty
snapshots allowed (an empty snapshot is a cheap "the task ran" receipt; suppressing them
loses that signal).

vault (VM 107) built to spec — Ubuntu Server 24.04 standard install, q35 + OVMF, 3 vCPU with
CPU units 50, 12 GB with ballooning off, 80 GB scsi0 on the NVMe with discard + SSD emulation
+ iothread, vmbr1 tag 30, qemu-guest-agent. Configured DHCP in-OS with a Kea reservation
pinning 10.0.30.40, following cantina's Phase 4 deviation rather than the static-in-OS
pattern, so the lease registers the name in Unbound. FQDN verified resolving before any
service went on the box.

### 2026-07-24 — vault's self-hostname resolved to 127.0.1.1

#### Issue
`getent hosts vault` returned `127.0.1.1`, even though Unbound resolved the FQDN correctly
and `dig` returned 10.0.30.40.

#### Root cause
The Ubuntu installer writes `127.0.1.1 <fqdn> <host>` into `/etc/hosts`, and nsswitch's hosts
order (`files dns`) means that line shadows DNS for the machine's own name. Harmless on a
workstation. Not harmless on a service VM: anything that resolves its own hostname to pick a
bind or advertise address would take loopback — the failure class where the web UI loads fine
but ML and WebSocket calls silently fail, and a counter reads zero while nothing happens.

#### Fix
Retargeted the `127.0.1.1` line to 10.0.30.40 — the pinned Kea reservation, so it cannot go
stale — leaving `127.0.0.1 localhost` and the IPv6 block intact.

#### Learning
On any service VM that advertises its own name, correcting the installer's `127.0.1.1` line
to the reserved IP is a standard build step, not a troubleshooting step. Fix it before the
service is installed and the whole failure class never happens.

### 2026-07-24 — sudo/runuser reject a numeric-only UID for the NFS write test

#### Issue
`sudo -u '#3001'` and `runuser -u '#3001'` both failed with "unknown user", blocking the
write test that was supposed to prove the identity chain.

#### Root cause
No local passwd entry for UID 3001 on vault — the identity was defined only on the NAS. Both
tools need a *resolvable account* to switch to, not merely a number; the kernel is happy with
the number, the userspace tooling is not.

#### Fix
Created a matching local service account (`groupadd -g 3001 immich`; `useradd -u 3001
-g 3001 -M -N -s nologin immich`) and re-ran the test as `sudo -u immich` → WRITE-OK /
DELETE-OK, with root still denied.

#### Learning
Mirror the NAS service UID as a local nologin account on the consumer VM. It makes the
identity legible on both ends of the wire, and it is the same UID the containers run as via
compose's `user:` key — one number, three places, no translation layer to get wrong.

### 2026-07-24 — Docker GPG key fetch failed on paste

#### Issue
The `curl` for Docker's signing key produced `(23) failure writing output`, then left an HTML
error page saved where the key should have been.

#### Root cause
A multi-line paste with a backslash continuation flattened in the terminal, detaching the
`-o` path from the URL — compounded by needing sudo to write under `/etc/apt/keyrings`.

#### Fix
Ran the fetch as a single unbroken line
(`sudo curl -fsSL ... -o /etc/apt/keyrings/docker.asc`) and verified the result: `head -1`
shows the PGP BEGIN line, and the file is non-zero (3817 bytes, mode 0644).

#### Learning
Paste multi-line command blocks as single lines. And verify keyring files are non-empty and
actually contain a key *before* `apt update`, rather than discovering it later as a NO_PUBKEY
error two steps removed from the cause.

### 2026-07-24 — Immich v3.0.3 deployed and configured

Docker CE installed from the official repo, and the Immich v3.0.3 stack brought up: four
containers healthy. The database image is the required pgvector + VectorChord build — face
clustering runs as vector-index queries in the database, so stock Postgres is not a
substitution. The `user: "3001:3001"` identity chain holds under the running container, with
no EACCES on the NFS paths.

Admin settings applied before any photo was imported:

- **Storage template** `{{y}}/{{y}}-{{MM}}/{{filename}}` enabled **pre-import** — enabling it
  after a bulk import would rename the entire library across NFS.
- **Transcode policy** set to "higher than target res or not accepted" with H.264 + HEVC
  video, AAC audio, and MOV container all marked accepted, so the 3,119 iPhone HEVC/.mov
  videos are left untouched. The default accepts H.264 only and would have transcoded the
  entire video library to no benefit.
- **ML job concurrency** capped at 2 for Smart Search and Face Detection, preserving CPU
  headroom for tarkin on the shared 7700T.
- **Database backup** 02:00 daily, keep 14.
- **Egress:** Version Check on, Map on, Google Cast off (ADR 0017).
- **Hardware acceleration** disabled — CPU-only ML by design.

### 2026-08-05 — immich_server did not restart after a vault reboot — automount/Docker boot race

#### Issue
The Immich web UI was unreachable. `docker compose ps` showed only three containers —
`immich_server` absent from the running set — and `docker ps -a` revealed it `Exited (128)`
seven days prior. The last database dump in `/mnt/photos/backups` was dated 2026-07-28,
matching `uptime`, which showed vault up 7 days.

#### Root cause
vault rebooted on 2026-07-28. Exit code 128 means the container runtime failed *before* the
application started, which puts the fault below Immich rather than inside it.
`immich_server` is the only container with a bind mount to `/mnt/photos`, and that path is an
`x-systemd.automount` trigger that had not yet fired at Docker start — the bind mount source
did not exist, so the runtime aborted. The other three containers have no NFS dependency
(Postgres on local disk per ADR 0015, ML on a Docker volume, Valkey in memory), so
`restart: always` brought them back normally. That is why the stack looked three-quarters
healthy instead of obviously broken.

#### Fix
A systemd override on `docker.service` adding `RequiresMountsFor=/mnt/photos`, which orders
Docker after the automount unit and pulls it in as a dependency instead of assuming it is
already up. Verified by a deliberate reboot: all four containers healthy, no manual
intervention.

#### Learning
`x-systemd.automount` defers mounting until first access — and that deferral is exactly what
prevents the silent-write-into-an-empty-mountpoint failure `nofail` would allow (ADR 0015).
The mount refused correctly rather than letting Immich write to local disk and split the
library. The cost is a boot-ordering race for anything that bind-mounts the deferred path,
and the price of fixing it is one line of unit override: any container or service depending
on an automounted path needs an explicit `RequiresMountsFor` ordering dependency. This
applies to every future service VM consuming holocron over NFS.

One diagnostic misstep worth recording alongside it: `ls /mnt/photos` as an unauthorized user
returns `Permission denied`, which is root squash working as designed — not a mount failure.
Read as a mount test, it points at the wrong layer entirely. Mount health has to be tested as
the authorized identity (`sudo -u immich`) and confirmed structurally with `findmnt -t nfs4`;
a generic success/fail test conflates "not mounted" with "mounted but access denied."

### 2026-08-05 — Root-cause investigation: 2026-07-28 outage and 2026-08-02 drive fault are unrelated

#### Method
Ran the known-good comparison first — `uptime` across every VM and the host — rather than
opening vault's journal. One command separated host-level from VM-level, and it came back
host-level, which made a full journal dive on vault unnecessary. Cheapest test with the
highest discriminating power, before the expensive one.

#### Finding 1 — executor rebooted deliberately
`journalctl --list-boots` shows boot -1 ending 2026-07-28 22:32:01 and boot 0 beginning
22:34:26 — a two-minute gap, consistent with a restart rather than a power outage. The tail
of boot -1 shows `pvescheduler` ticking normally right up to 22:32:01 and then simply ending:
no kernel panic, no OOM kill, no call trace, no thermal or I/O fault. The Proxmox task log
records "Bulk start VMs and Containers" by `root@pam` at 22:34:37 with all seven VMs starting
OK — normal post-boot autostart. A grep for `Automatic-Reboot` across `/etc/apt/apt.conf.d/`
returned nothing, and apt activity around the date was download and list-refresh only (two
manual "Update package database" tasks), so no unattended-upgrade reboot. All VMs except 107
show ~7 days uptime, confirming the event was host-level.

#### Finding 2 — the drive fault is five days later and causally unrelated
`dmesg -T` on archives timestamps the failures at Sun Aug 2 00:00:05 and 00:03:23 — "critical
medium error … Sense Key: Medium Error … Unrecovered read error" against specific sectors on
`sdd`. That is physical media failure reported by the drive's own firmware, not a cable,
controller, or timeout fault. 00:00 Sunday is the start of the scheduled weekly scrub: the
scrub systematically read every block, hit the unreadable sectors, reconstructed them from
parity (1020K repaired in 8m37s with 0 errors), and ZFS faulted the device once it crossed
its error threshold. `zpool status -v` confirms raidz2-0 fully ONLINE with zero errors and
raidz2-1 DEGRADED with one FAULTED device (34 read / 2 cksum, "too many errors"), every other
member at zero, and pool-level READ/WRITE/CKSUM all 0.

#### Learning
The scrub found a latent media defect that was otherwise silent. That is the argument for
scheduled scrubs stated concretely rather than as received wisdom: blocks nobody reads hide
their bad sectors until something finally needs them, and that something is usually a rebuild
— the worst possible moment to discover a second unreadable drive.

Two incorrect hypotheses were formed earlier in this investigation — that the NFS export had
broken, and that the drive failure had destabilised the host — and both trace back to a
single badly designed diagnostic, the one recorded in the entry above, that conflated "not
mounted" with "mounted but access denied to the calling user." Correct method: test mount
health as the authorized identity and confirm with `findmnt -t nfs4`, and establish event
ordering from timestamps before asserting causation.

### 2026-08-11 — holocron drive `sdd` failed and was replaced — pool resilvered clean

#### Issue
holocron DEGRADED. `sdd` — an HGST HUS724040ALS640, serial PBG7ZLHX, on the SAS3008
controller — FAULTED with 34 read and 2 checksum errors.

#### Root cause
Genuine media failure. `dmesg -T` showed "critical medium error … Sense Key: Medium Error …
Unrecovered read error" against specific sectors: reported by the drive's own firmware, so not
a cable, controller, or timeout fault. The errors surfaced at 00:00:05 on Sun 2026-08-02, the
start of the scheduled weekly scrub — the scrub read every block, hit the unreadable sectors,
reconstructed them from parity (1020K repaired in 8m37s with 0 errors), and ZFS faulted the
device once it crossed threshold. raidz2-0 stayed fully ONLINE; only raidz2-1 degraded, and it
still held one disk of redundancy. Pool-level READ/WRITE/CKSUM were 0 throughout — no data
lost.

#### Fix
Replaced with an HGST HUS72404CLAR4000, serial PEJSRYWX — the EMC-branded variant of the same
Ultrastar 7K4000. Sector format verified **before** installing: `sg_readcap --long` reported a
logical block length of 512 bytes. That check is the critical one, because ex-array enterprise
drives — EMC-branded ones especially — are frequently 520-byte formatted and unusable by ZFS
without a multi-hour `sg_format`. Cold-swapped into the same physical slot; the new disk
inherited the device name `sdd`. ZFS created a `replacing-2` vdev holding the absent old
device as UNAVAIL (identifiable only by GUID/partuuid) alongside the new device ONLINE and
resilvering. The resilver completed with 0 errors and the pool returned to ONLINE.

#### Learning
(a) Scheduled scrubs surface latent media defects that are otherwise silent until the data is
actually needed — this incident is the concrete argument for them rather than the received-
wisdom version. (b) Always verify logical block size on a used enterprise SAS drive before
installing it; finding 520-byte sectors afterwards costs hours. (c) A cold swap into the same
slot is sufficient — ZFS resolves the replacement by GUID, and the new drive taking over the
old device name is expected, not a sign of confusion.

### 2026-08-11 — Pilot import passed

Roughly ten hand-picked assets uploaded with `immich-go upload from-folder
--into-album=pilot`. Verified image fidelity, capture dates, RAW, slo-mo, and Live Photo
pairing — the HEIC and MOV halves merged into a single motion asset with the still as cover.

Worth recording what the pilot did *not* cover: `from-folder` does not read the takeout CSVs,
so album reconstruction was untested by the pilot and had to be validated separately during
the bulk import.

### 2026-08-13 — Bulk import failed twice before succeeding — transcode storm, then client-side OOM

#### Issue (run 1)
immich-go stalled at ~23% — 3,975 of 17,436 assets processed, 13,430 pending — with 22 server
errors totalling 5.3 GB. That averages ~240 MB per failure, i.e. large videos.

#### Root cause (run 1)
A transcode storm. The transcode policy was "higher than target resolution **or** not in an
accepted format" — two conditions joined by OR. HEVC was marked an accepted codec, but Target
Resolution had been left at 720p, so every 4K iPhone video tripped the resolution clause and
the codec setting never got a chance to apply. Compounding it, Threads was 0, which Immich
documents as "maximizes utilization": each transcode job consumed all 3 vCPUs, starving upload
handling, hashing, and thumbnailing. Large-video uploads timed out server-side and immich-go
gave up feeding the server.

#### Fix (run 1)
Transcode policy set to "Don't transcode any videos", and made permanent — the primary client
is the iOS app, which decodes 4K HEVC in hardware, and Immich v3 transcodes on the fly when a
browser needs it. Target Resolution corrected 720p → original; Threads 0 → 2. Queued transcode
jobs cleared.

#### Issue (run 2)
With transcoding fixed, the run reached 83% and then crashed falcon at 97% memory usage.

#### Root cause (run 2)
Client-side memory exhaustion. immich-go holds discovery state for the whole tree in memory —
paths, hashes, and album membership cross-referenced from the takeout CSVs — and 145 GB across
~17,400 assets exceeded falcon's RAM.

#### Fix (run 2)
Imported each Part folder separately (~35 GB each, roughly a quarter of the working set), then
ran a final pass against the parent folder. By then every asset was already on the server, so
that pass performed no uploads — it only read the manifests and applied album membership by
checksum, a far lighter operation. Server job queues (thumbnails, metadata, smart search, face
detection, transcode) were left PAUSED throughout, so the server spent its CPU accepting files
rather than processing them; all processing then ran afterwards in one uninterrupted stretch.
immich-go pauses these queues by default during upload.

#### Result
Complete import, albums reconstructed correctly, 148.9 GiB stored — above the 145.35 GB source
takeout, as expected once Immich generates its derivatives.

#### Learning
(a) An OR-joined transcode policy means **both** clauses have to be correct; an accepted-codec
list cannot save you from a low target resolution. (b) Threads=0 means unlimited, not default,
and unlimited on a 3-vCPU VM sharing a host with the router is a starvation risk. (c)
immich-go's memory footprint scales with tree size, so a large takeout should be imported in
chunks with a final parent-folder pass to apply album membership. (d) Pausing server job
queues during bulk ingest separates the two workloads instead of letting them compete for the
same CPU.

### 2026-08-13 — Phone cutover complete

iCloud Photos disabled with "Remove from iPhone"; the device library dropped from ~145 GB of
optimized placeholders to ~12 GB of local originals.

The Immich mobile app required an explicit iOS camera-roll permission grant — without it the
in-app backup toggle appears configured and silently does nothing, so the permission has to be
verified separately from the setting. Backup enabled on all device photos, and a newly taken
photo uploaded to the server immediately, confirming the daily loop end to end.

iCloud+ is retained until its 2026-09-09 renewal so that geographic separation is maintained
while the Pi 5 offsite appliance is completed — cancellation remains gated on
`holocron/photos` replicating offsite (ADR 0016).

### 2026-08-14 — echo-base built, carbonite seeded, ADR 0016 cancellation gate satisfied

The offsite appliance was built and the first pull completed, which closes the last
structural dependency in Phase 5. Detail lives in
[ADR 0010's implementation amendment](../decisions/0010-offsite-replication-mechanism.md)
and [runbooks/offsite-replication.md](../runbooks/offsite-replication.md); this entry is the
build narrative.

**echo-base** — Raspberry Pi 5 on Ubuntu Server 24.04 LTS (arm64), flashed with Raspberry Pi
Imager using cloud-init customization for hostname, user, SSH key, key-only auth, and no
WiFi. It lives at mom's house in NJ, behind senate, on ethernet with a DHCP address. Not
static in-OS, and deliberately so: the appliance is meant to stay location-portable, and its
durable identity is its tailnet address rather than any LAN IP. Timezone corrected from the
Imager default (`Etc/UTC`) to `America/New_York` — see the timezone entry below, which turned
out to matter more than it looks.

**carbonite** — a single-disk encrypted ZFS pool on the 6 TB drive, addressed by
`/dev/disk/by-id` rather than `/dev/sdX`, because enumeration order is not stable and is
especially unstable over USB. Created with `ashift=12` (permanent, 4K alignment),
`compression=lz4`, `atime=off`, `encryption=aes-256-gcm`, `keyformat=raw`, and
`keylocation=file:///root/.carbonite.key` — 32 bytes from `/dev/urandom`, mode 400. A keyfile
rather than a passphrase because the appliance has to come back on its own after an
unattended power cut with nobody in the house to type anything. The drive sits in a USB-C
4-bay enclosure (ASMedia ASM235CM bridge, VIA Labs internal hub) that negotiated UAS.

**Network path** — no new firewall rules, no port forwards, no static routes. echo-base joins
the tailnet and accepts phantom's advertised `10.0.30.0/24` route with
`tailscale up --accept-routes`; Linux clients ignore advertised subnet routes unless
explicitly opted in, unlike iOS and macOS. IP forwarding is enabled via
`/etc/sysctl.d/99-tailscale.conf` — not needed for the pull itself, pre-staged for the
planned site-to-site routing onto mom's LAN. Key expiry is disabled on this node: an
unattended box carrying a ~180-day node key expiry is a silent outage timer. Verified with
`ping 10.0.30.20` returning `ttl=63` — one hop, phantom forwarding. This is the same path
that will keep working after the lab relocates to NY, so the appliance needs no
reconfiguration then.

**pibackup** — the least-privilege service account on archives: TrueNAS local user, UID 3002,
own primary group, password disabled, no SMB, no UI access, no API keys, home at
`/mnt/holocron/pibackup`. Shell is `/usr/bin/bash` and that is required rather than sloppy —
syncoid runs `zfs send` over SSH and a `nologin` shell refuses the connection outright. The
privilege restriction comes from ZFS delegation, not from removing the shell. Two SSH public
keys are authorized: echo-base's user key for interactive troubleshooting, and echo-base's
root key for the scheduled job. Delegation was set with `sudo zfs allow` from an admin shell
(`truenas_admin` alone is insufficient — delegation requires root):
`zfs allow pibackup send,snapshot,hold,bookmark,mount holocron/photos`, and the same for
`holocron/media`. Verified reading back as
`Local+Descendent permissions: user pibackup bookmark,hold,mount,send,snapshot`.
Deliberately absent: `receive`, `create`, `destroy`, `rollback`. A fully compromised Pi can
read the source and do nothing else — it cannot encrypt, delete, or corrupt holocron. That is
the ransomware-resistance property of ADR 0010 made concrete rather than asserted.

**Replication** — syncoid 2.3.0 (sanoid package) on echo-base, pull-based:

```
sudo syncoid --no-privilege-elevation --sshkey=/root/.ssh/id_ed25519 --no-sync-snap \
  pibackup@10.0.30.20:holocron/<dataset> carbonite/<dataset>
```

`--no-privilege-elevation` because syncoid otherwise attempts sudo on the source, and
pibackup has none by design and needs none — the `zfs allow` delegations cover exactly what
it does. The send is **non-raw** (no `-w`): holocron is unencrypted (`encryption off`), so
raw send does not apply. Data travels as plaintext inside the SSH-over-Tailscale tunnel and
is encrypted at rest by carbonite's own key on arrival, so a stolen Pi yields ciphertext
decryptable only by a key that never left it. Root uses a dedicated keypair
(`/root/.ssh/id_ed25519`, no passphrase, comment `echo-base-root-backup`) rather than
borrowing the user's key — that separates machine identity from personal identity, so either
can be rotated or revoked independently and the scheduled job survives a change to the user
account.

**Schedule** — systemd timers rather than cron, because run history is queryable via
`journalctl`, the next fire time is inspectable, runs are testable on demand, and
`Persistent=true` catches up a run missed during a power cut.
`syncoid-media.{service,timer}` at 04:00 daily and `syncoid-photos.{service,timer}` at 04:30
daily; both `Type=oneshot` with `After=network-online.target tailscaled.service`, since
firing before the tunnel exists would just fail with a connection error.

**Monitoring** — a healthchecks.io dead-man's switch, one check per dataset, pinged from
`ExecStartPost` so it fires only on `ExecStart` success. A failed run therefore produces
silence, and silence is what raises the alert. Period is 1 day.
TODO(cole): record the configured grace period.

### 2026-08-14 — Pi booted but was unreachable — a stale bootable OS on a USB drive won the boot order

#### Issue
Solid green LED and ethernet link lights, but no DHCP lease under the expected hostname and
no SSH. The cloud-init files on the SD card were present and correct.

#### Root cause
One of the drives in the USB enclosure carried a bootable OS, and the Pi 5 bootloader
preferred it over the SD card. The machine came up running an unrelated operating system
with none of the configured identity — hence no expected hostname, no configured user, and
no authorized key.

#### Fix
`wipefs -a` plus `sgdisk --zap-all` against the drive by its `by-id` path. Both are needed:
`wipefs` clears filesystem signatures, `sgdisk --zap-all` also removes the backup GPT header
at the end of the disk, which `wipefs` alone can leave behind.

#### Learning
Every easy-to-check indicator read healthy — power, link, cloud-init files on the card —
while the actual state was one layer deeper. Same silent-failure shape as the Phase 3
zero-upstream DNS incident: the cheap indicators agreed with each other and all of them were
answering a different question than the one being asked.

### 2026-08-14 — pibackup was auto-assigned `sudo` as its primary group

#### Issue
The TrueNAS Add User form auto-populated `sudo` as the new account's primary group. Caught
during a pre-save field-by-field review, before the account was created.

#### Root cause
Form default, not an explicit choice.

#### Fix
Changed to its own primary group before saving. The home directory needed correcting in the
same pass — the form defaulted to `/var/empty`, which is shared and non-writable — and was
set to `/mnt/holocron/pibackup`.

#### Learning
A sudo-capable account whose private key lives on a Pi in another house would have inverted
the entire security model of ADR 0010: the whole point is that the offsite credential cannot
modify the source. Reviewing the form field by field caught what a glance would have missed,
precisely because every *other* setting on that form read correctly locked-down.

### 2026-08-14 — `bash: zfs: command not found` over SSH

#### Issue
`zfs` ran fine in an interactive session on archives but was not found when invoked over SSH
the way syncoid invokes it.

#### Root cause
TrueNAS spawns non-interactive shells that do not source `/etc/profile`, and the ZFS binaries
live in `/usr/sbin`, which is absent from the resulting PATH. Adding a PATH line to the
user's `.bashrc` did **not** fix it — bash does not source `.bashrc` for non-interactive
sessions here either.

#### Fix
Appended a PATH entry to `/etc/environment` on archives. Verified with the exact
non-interactive form syncoid uses rather than an interactive login:
`ssh -i <key> pibackup@10.0.30.20 "which zfs"` → `/usr/sbin/zfs`.

#### Learning
Test the environment in the same shape the caller will use it. An interactive login proves
nothing about a non-interactive one, and this failure is invisible until the real job runs.
Caveat worth carrying forward: `/etc/environment` edits may not survive a TrueNAS update —
that is the first thing to re-check if replication breaks after an upgrade.

### 2026-08-14 — `Permission denied (publickey)` under sudo, but not interactively

#### Issue
`ssh pibackup@10.0.30.20` worked interactively; `sudo syncoid ...` failed with
`Permission denied (publickey)`.

#### Root cause
sudo runs SSH as root, and root has its own `~/.ssh`. It could not see the user's key at all.

#### Fix
Interim: pass the key explicitly with `--sshkey=`. Final: a dedicated root keypair on
echo-base, with its public half added to pibackup's authorized keys.

#### Learning
The final fix is better than the interim one for a reason beyond making the error go away —
a machine-owned key for a machine-run job keeps the scheduled pull independent of the
personal account, so either identity can be rotated without breaking the other.

### 2026-08-14 — Two machines, two wrong timezones, and a backup that would have silently lagged a day

#### Issue
The Pi shipped on `Etc/UTC` (the Imager default) and archives was on `America/Los_Angeles`
(the TrueNAS install default). The lab is Eastern.

#### Root cause
Two independent installer defaults, neither noticed, on two machines whose jobs depend on
each other.

#### Fix
Both set to `America/New_York`. systemd `OnCalendar` is interpreted in system-local time, so
the replication timers followed automatically with no edits to the unit files.

#### Learning
This is the entry worth re-reading. Nothing would have errored. The replication timers would
have fired hours before each night's snapshot existed and quietly replicated the *previous*
day's state forever — a backup that runs, reports success, and is permanently one day stale.
Cross-machine schedule dependencies are only correct if the clocks agree, so verify timezone
alignment whenever one machine's job depends on another machine's job. Second consequence,
found in passing: archives' log timestamps were three hours off from the rest of the lab,
which would have corrupted event correlation in Phase 6 the moment Wazuh started ingesting
them.

### 2026-08-14 — Sequencing: why offsite replication was deferred until after the import

Recorded as a decision rather than a defect. The offsite replication was deliberately held
back until the Immich bulk import and reconciliation were complete. Replicating mid-migration
would have captured a partial library, required a full re-replication afterwards, and — the
real risk — could have been mistaken for a satisfied cancellation gate while the thing it was
supposed to protect did not yet exist in full. The appliance itself was built in parallel
with the migration; only the final pull was gated.

---

## Deferred / follow-ups

- **Nextcloud, or a lighter file-sync option** — revisit at Phase 6 planning, on the ADR 0014
  triggers (external share links, an auth-log-rich app target, or SMB proving insufficient).
- **Wazuh rule: no new DB dump in 48h** — Phase 6. The dumps are currently unmonitored;
  interim mitigation is a monthly manual check (ADR 0016 known gap).
- **Monitoring gap: Proxmox alert mail is not being delivered** — postfix on executor cannot
  reach the configured Gmail address (connect timed out / network unreachable to
  `gmail-smtp-in`, messages deferred), so Proxmox alert mail has not been arriving at all.
  The 2026-08-02 pool-degraded alert was never received, and the fault was found manually
  days later. Fix SMTP delivery — a relay with an app password, or an alternative
  notification target. A Phase 6 monitoring input, and arguably the most important finding of
  the 2026-08-05 investigation.
- **`pvescheduler` replication-state noise on executor** — `pvescheduler` logs `replication:
  invalid json data in /var/lib/pve-manager/pve-replication-state.json` every 60 seconds, and
  has done since at least 2026-07-19. No PVE replication jobs are configured, so it is
  harmless in itself — but it floods the journal and would bury real signal during a future
  incident. Reset the state file. The "before the bulk import" deadline this was originally
  written against has passed, so confirm the current state and clear it.
- **Immich patch-cadence automation** — candidate for Phase 8; Immich does not backport, so
  this box moves faster than the rest of the lab (ADR 0014).
- **`runbooks/immich-restore.md`** — to be written at phase close. Must state plainly that a
  database dump alone is not a restore: it takes the dump *plus* `UPLOAD_LOCATION`, together.
- **A minority of assets carry incorrect timeline dates** — Immich falls back to file mtime
  (2026-08-04, the extraction date) when EXIF `DateTimeOriginal` is absent. The affected
  population is predictable: screenshots, screen recordings, re-exported or edited images,
  Snapchat-sourced files, and some videos whose timestamp lives in QuickTime container
  metadata rather than EXIF. Screenshots have been archived out of the main timeline, which
  resolves most of the visible impact. The residual is **accepted as-is** — Immich cannot
  infer a date that does not exist in the file, and per-asset correction is not worth the
  effort at this volume. Revisit only if a specific album or year proves materially wrong.
- **Immich v3.1.0 is available** — deliberately *not* applied during the migration; the
  deployment stays pinned to `v3.0.3` in `.env`. Updating is a post-phase decision requiring a
  read of the release notes, per the ADR 0014 patch-cadence consequence.
- **Free Up Space — now unblocked, still not run** — Immich's local-asset reclamation shows a
  review screen and moves assets to iOS Recently Deleted rather than deleting outright, but
  it is the irreversible step in the cutover. It was gated on the Pi 5 replicating, which is
  satisfied as of 2026-08-14. Running it is now a judgement call rather than a blocked one;
  the conservative order is to let the two-week soak finish first.
- **`holocron/configs` was never created** — so the OPNsense XML export and the switch config
  have no offsite copy and live only in the local, gitignored `config-backups/`. ADR 0010's
  Consequences listed this as a requirement and it is unmet; the replication set currently
  carries `holocron/photos` and `holocron/media` only.
  TODO(cole): open as a tracked follow-up, or accept explicitly.
- **echo-base is a single point of failure by design** — one disk, no redundancy, at a remote
  site (ADR 0010 amendment). A detected checksum error is answered by re-replicating from
  holocron, which is fine while holocron is healthy — but it does mean the offsite tier has
  no independent recovery path of its own. Revisit only if the offsite copy ever becomes the
  primary during a relocation.
