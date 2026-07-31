# 0015 — Immich data placement and read-write NFS semantics
**Status:** Accepted (Phase 5, 2026-07-24)
**Context:** Immich has two storage classes with opposite requirements. Postgres needs POSIX
file locking that NFS does not reliably provide — the failure mode is corruption, and
upstream explicitly unsupports network shares for `DB_DATA_LOCATION`. The asset library
(`UPLOAD_LOCATION`) is bulk data, which the Phase 2 rule places on holocron (OS and
application on NVMe, data on the pool). `UPLOAD_LOCATION` is six subfolders Immich moves
files between with `rename(2)`, which fails EXDEV the moment the tree spans filesystems.
**Decision:** `DB_DATA_LOCATION=/opt/immich/postgres` on vault's local NVMe-backed virtual
disk. `UPLOAD_LOCATION=/mnt/photos` on a dedicated dataset `holocron/photos`
(recordsize=1M, atime=off), whole tree on one filesystem. Mount options:
`nfs4 rw,hard,nconnect=4,noatime,_netdev,x-systemd.automount`. Identity: TrueNAS user and
group `immich` at UID/GID 3001; dataset 0770 `immich:immich`; NFS export scoped to
10.0.30.40/32 with maproot left empty, so root squash stays on and root on vault arrives as
`nobody` and cannot touch the dataset; containers run `user: "3001:3001"`. Process UID =
NFS wire UID = dataset owner — one number end to end.
**Alternatives:** Postgres on NFS (rejected — unsupported upstream, and the failure mode is
silent corruption); `ro,soft` as on cantina (rejected — a soft mount timing out on a WRITE
returns EIO without the client knowing whether the bytes landed, which is silent divergence
between Immich's database and its library; cantina's rationale, that a failed read is a
failed playback and nothing is lost, does not transfer to a write path); `nofail` instead of
`x-systemd.automount` (rejected — nofail plus a down NAS means Docker starts and Immich
writes into an empty local mountpoint, silently splitting the library, whereas automount
makes the mountpoint a kernel trigger that blocks instead); maproot `immich`/`immich` with
containers as root (held as the fallback lever, below).
**Consequences:** A stalled mount hangs Immich and nothing else. The ADR 0009 deadlock
analysis does NOT apply here: vault↔archives is same-subnet on vmbr1 and never transits
tarkin, and vault sits on no path archives depends on. Falsifier: if a vault NFS stall is
ever observed to affect anything outside vault, that premise is wrong and this design
reopens. Identity falsifier: EACCES on `/data` paths in the `immich_server` log means the
UID chain broke; the fallback is maproot `immich`/`immich` plus containers as root — writes
still land as 3001, but container-escape protection is lost. Which lever was used is
recorded. (Lever used: none — UID-match with root squash verified working; see the Phase 5
log.)
