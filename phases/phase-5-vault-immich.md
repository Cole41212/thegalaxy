# Phase 5 — vault (Immich) & the iCloud Migration

**Goal:** Replace a paid iCloud+ subscription with a self-hosted photo platform: build vault
(VM 107), run Immich against a dedicated ZFS dataset over read-write NFS, extract ~145 GB of
owned originals from Apple, and cut the phone over — without ever holding fewer than three
copies of the library. Along the way: the platform choice (ADR 0014), the data-placement and
rw-NFS semantics (ADR 0015), the deployment/extraction/backup model (ADR 0016), and the
access and TLS posture (ADR 0017).

**Status:** 🚧 In progress — build and deployment complete (through Stage 3); extraction,
import, cutover, and the offsite gate pending

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
| Takeout extracted on falcon | ⬜ | 4 archives → staging |
| Pilot import passed | ⬜ | Small batch first — dates, album membership, Favorites check |
| Bulk import | ⬜ | `immich-go from-icloud` |
| ML processing complete | ⬜ | Smart Search, face detection, OCR backfill |
| Reconciliation vs source | ⬜ | Final asset count against the takeout manifest; note shared-album exclusions |
| Screenshots archived | ⬜ | Own album → Archive, during import |
| LTE verification | ⬜ | Immich reachable over the tailnet off home WiFi |
| DB dump verified | ⬜ | First 02:00 dump present in `UPLOAD_LOCATION/backups` |
| vzdump enrolled | ⬜ | Auto-enrolled via the job's "all except" selection (ADR 0009) — confirm |
| Snapshot fired | ⬜ | First 03:00 snapshot on `holocron/photos` |
| Phone cutover (iCloud Photos off) | ⬜ | Only after import + reconciliation pass |
| Two clean weeks | ⬜ | Auto-backup running, no gaps |
| Pi 5 replicating `holocron/photos` | ⬜ | ADR 0010 — the cancellation gate |
| iCloud+ cancelled | ⬜ | Gated on the line above |
| Cleanup | ⬜ | Revoke the API key, restore falcon's power settings |

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
| Transcode policy | "Higher than target res or not accepted"; accepted video H.264 + HEVC, audio AAC, container MOV | The default accepts H.264 only, which would transcode all 3,119 iPhone HEVC/.mov videos for nothing |
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

---

## Deferred / follow-ups

- **Nextcloud, or a lighter file-sync option** — revisit at Phase 6 planning, on the ADR 0014
  triggers (external share links, an auth-log-rich app target, or SMB proving insufficient).
- **Wazuh rule: no new DB dump in 48h** — Phase 6. The dumps are currently unmonitored;
  interim mitigation is a monthly manual check (ADR 0016 known gap).
- **Immich patch-cadence automation** — candidate for Phase 8; Immich does not backport, so
  this box moves faster than the rest of the lab (ADR 0014).
- **Screenshots** — route into their own album and Archive them during import, so they stay
  out of the timeline without being deleted.
- **`runbooks/immich-restore.md`** — to be written at phase close. Must state plainly that a
  database dump alone is not a restore: it takes the dump *plus* `UPLOAD_LOCATION`, together.
- **Reconciliation** — final asset count against the takeout manifest, with any shared-album
  exclusions noted explicitly rather than absorbed as a discrepancy.
