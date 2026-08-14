# Backup Strategy

Goal: a real 3-2-1 posture — at least 3 copies, on 2 media types, 1 offsite — that
distinguishes *redundancy* (survives a drive) from *backup* (survives the box) from
*disaster recovery* (survives the site).

## Layers
### 1. ZFS RAIDZ2 — redundancy, not backup
The `holocron` pool (2× 6-drive RAIDZ2) tolerates 2 simultaneous drive failures per vdev.
This protects against drive failure only — it is NOT a backup (ransomware, fat-finger
deletes, and pool corruption all replicate instantly).

### 2. Local backups — convenience + space (same site)
- **Service VM backups:** nightly Proxmox vzdump → TrueNAS `vm-backups` dataset (NFS,
  soft mount). Large, cheap space on RAIDZ2. The job's "all except 100/102" selection
  auto-enrolls future VMs — phantom was picked up automatically in Phase 3.
- **Infrastructure VM backups (tarkin 100, archives 102):** nightly vzdump → `ssd-vmstore`
  (local, network-independent). The NAS path depends on tarkin for routing and on archives
  for the storage itself — backing them up over it deadlocks (see decisions/0009 and the
  2026-06-23 incident in the Phase 2 log).
- **archives must not back up to itself.** Its TrueNAS config + 32 GB boot vdisk back up to
  the NVMe/SSD, never to the pool it serves.
- **TrueNAS snapshots:** periodic ZFS snapshots on `media`, `files`, and `photos` (03:00
  daily, keep 2 weeks — Phase 5) for quick rollback of accidental changes; weekly scrub for
  integrity. The scrub is not ceremonial: the 2026-08-02 run found a latent unreadable
  sector on a SAS drive and faulted the device before the bad block was ever needed for a
  rebuild (phases/phase-5-vault-immich.md).
- **Immich database dump (vault):** Immich's own logical dump, 02:00 daily keep 14, written
  into `UPLOAD_LOCATION/backups` on `holocron/photos` — so it lands off the VM, inside the
  03:00 snapshot, and inside the offsite replication set automatically. vzdump is only
  crash-consistent and upstream warns against copying a live `DB_DATA_LOCATION`, so the
  logical dump is database authority (ADR 0016). **A dump alone is not a restore:** it takes
  the dump *plus* `UPLOAD_LOCATION`, together.
- Reality check: this lives on the same host. If executor dies, none of it helps. It is
  fast-restore convenience, not disaster recovery.

### 3. Offsite — disaster recovery (the real backup)
**Operational since 2026-08-14** (ADR 0010, Phase 5). Day-to-day procedures live in
[runbooks/offsite-replication.md](../runbooks/offsite-replication.md).

- **Appliance:** `echo-base` — Raspberry Pi 5, Ubuntu Server 24.04 LTS (arm64), at a family
  member's residence. Ethernet, DHCP on that site's LAN, reachable **only** over Tailscale —
  no port-forwarding, no exposure, no inbound anything. Deliberately not static in-OS: the
  appliance is location-portable and its durable identity is its tailnet address.
- **Pool:** `carbonite` — single 6 TB disk, `ashift=12`, `compression=lz4`, `atime=off`,
  native ZFS encryption (`aes-256-gcm`), unlocked at boot from a keyfile on the Pi's SD card
  so the appliance recovers unattended after a power cut. Addressed by `/dev/disk/by-id`,
  never `/dev/sdX` — USB enumeration order is not stable.
- **Single disk is a choice, not a shortfall.** This tier's job is geographic separation and
  ransomware isolation; holocron's RAIDZ2 is the resilience layer. Checksumming and verified
  send/recv are kept, only self-healing is given up, and a detected error is answered by
  re-replicating from the intact source. Full reasoning in the ADR 0010 amendment.
- **Scope — what replicates:** `holocron/photos` (149 GB — the Immich library *and* its
  nightly logical DB dumps, so one stream carries both halves of a restore) and
  `holocron/media` (12.1 GB, replicated whole).
- **Scope — what does not, and why:** `holocron/vm-backups` (118 GB of vzdump output —
  already a backup; replicating backups-of-backups inverts the value of a capacity-limited
  tier), `holocron/files` (empty — Nextcloud deferred, ADR 0014), `holocron/crafty-backups`,
  `holocron/pibackup`. **Device configs are also absent** — `holocron/configs` was never
  created, so `config-backups/` has no offsite copy. See the open item in ADR 0010.
- **Mechanism (ADR 0010):** *pull-based* ZFS replication — `syncoid` on the Pi over
  Tailscale. The Pi initiates every transfer and holds the only credential, so a compromised
  lab cannot reach, let alone destroy, the offsite copy.
- **The credential:** `pibackup` on archives (UID 3002) — password disabled, key-only, no
  SMB, no UI, no API keys, and delegated exactly `send,snapshot,hold,bookmark,mount` on the
  two replicated datasets. `receive`, `create`, `destroy`, and `rollback` are deliberately
  withheld: a fully compromised Pi can read the source and nothing more. `--no-sync-snap` is
  used so syncoid works from the existing TrueNAS periodic snapshots rather than needing a
  destroy verb to clean up after itself.
- **Encryption in flight and at rest:** the send is non-raw, because holocron is
  unencrypted. Data crosses inside the SSH-over-Tailscale tunnel and is encrypted on arrival
  by carbonite's own key — a stolen Pi yields ciphertext decryptable only by a key that
  never left it.
- **Schedule:** systemd timers on echo-base — `syncoid-media` 04:00 daily,
  `syncoid-photos` 04:30 daily, both `Persistent=true` so a run missed during a power cut
  is caught up.
- **Monitoring:** healthchecks.io dead-man's switch, one check per dataset, pinged only on
  success — a failed run produces silence, and the silence raises the alert.
- **Prerequisites on archives:** TrueNAS snapshot tasks on the source datasets — done for
  `holocron/photos` and `holocron/media`. `holocron/media-keep` is no longer needed
  (superseded: media replicates whole); `holocron/configs` remains uncreated.
- The WD MyBook Live is explicitly NOT trusted as a host (EOL; the 2021 remote-wipe CVE).
  If ever used: Tailscale-only, never exposed.

### 4. Config backups — local + offsite, never public
OPNsense (tarkin) XML export and death-star switch config → `config-backups/` (local,
gitignored — NOT committed to the public repo; exports contain secrets) plus the offsite
copy. These make a bare-metal rebuild fast.

## Restore chain (disaster recovery runbook)
1. Reinstall Proxmox on executor; restore `/etc/network/interfaces` + host config.
2. Recreate/boot tarkin and archives from their NVMe/SSD backups (gets routing + NAS back).
3. Import the `holocron` pool (drives survive a host reinstall).
4. Restore remaining VMs from the `vm-backups` dataset. For vault, the vzdump restore is
   only the machine — Immich is not back until the latest logical dump from
   `holocron/photos/backups` is loaded *and* `UPLOAD_LOCATION` is remounted (ADR 0016).
5. Pull device configs from the local `config-backups/` folder and reapply. Note: configs
   are **not** currently in the offsite set (`holocron/configs` was never created), so this
   step depends on the local copy surviving — the one gap in the chain above.

For a restore *from* the offsite copy — including how to reach a file inside a carbonite
snapshot — see [runbooks/offsite-replication.md](../runbooks/offsite-replication.md).

## Schedule
| What | Frequency | Destination |
|---|---|---|
| vzdump service VMs (all except 100, 102) | nightly | TrueNAS `vm-backups` (NFS, soft mount) |
| vzdump infrastructure VMs (100, 102) | nightly | `ssd-vmstore` (local) |
| TrueNAS dataset snapshots | per dataset policy (`photos`: 03:00 daily, keep 2 weeks) | local pool |
| Immich logical DB dump (vault) | nightly 02:00, keep 14 | `holocron/photos/backups` — rides the snapshot and the offsite set |
| Offsite replication — `holocron/media` | nightly 04:00 | echo-base `carbonite/media` — syncoid pull over Tailscale |
| Offsite replication — `holocron/photos` | nightly 04:30 | echo-base `carbonite/photos` — syncoid pull over Tailscale |
| OPNsense + switch config export | per change | `config-backups/` (local, gitignored) |

## Accepted risk
The bulk media pool exists only on-site; only the irreplaceable subset goes offsite. A total
site loss means rebuilding the media library from source, which is acceptable.