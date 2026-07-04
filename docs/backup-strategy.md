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
  soft mount). Large, cheap space on RAIDZ2.
- **Infrastructure VM backups (tarkin 100, archives 102):** nightly vzdump → `ssd-vmstore`
  (local, network-independent). The NAS path depends on tarkin for routing and on archives
  for the storage itself — backing them up over it deadlocks (see decisions/0009 and the
  2026-06-23 incident in the Phase 2 log).
- **archives must not back up to itself.** Its TrueNAS config + 32 GB boot vdisk back up to
  the NVMe/SSD, never to the pool it serves.
- **TrueNAS snapshots:** periodic ZFS snapshots on `media` and `files` for quick rollback
  of accidental changes; weekly scrub for integrity.
- Reality check: this lives on the same host. If executor dies, none of it helps. It is
  fast-restore convenience, not disaster recovery.

### 3. Offsite — disaster recovery (the real backup)
- **Appliance:** Pi 5 + 6 TB USB HDD at a relative's house, reachable **only** over
  Tailscale (no port-forwarding, no exposure). Stood up in Phase 3.
- **Scope:** the *irreplaceable* subset — vault/Nextcloud files, all device configs, and
  selected media. Not the full ~29 TiB pool; that's the point of 3-2-1.
- **Mechanism:** ZFS `send | recv` over Tailscale if the Pi runs ZFS, else `restic`/`rsync`
  with encryption. Scheduled + monitored.
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
4. Restore remaining VMs from the `vm-backups` dataset.
5. Pull device configs from the local `config-backups/` folder or the offsite copy and reapply.

## Schedule
| What | Frequency | Destination |
|---|---|---|
| vzdump service VMs (all except 100, 102) | nightly | TrueNAS `vm-backups` (NFS, soft mount) |
| vzdump infrastructure VMs (100, 102) | nightly | `ssd-vmstore` (local) |
| TrueNAS dataset snapshots | per dataset policy | local pool |
| Offsite sync (files/configs/key media) | daily–weekly | Pi 5 + 6 TB via Tailscale |
| OPNsense + switch config export | per change | `config-backups/` (local, gitignored) |

## Accepted risk
The bulk media pool exists only on-site; only the irreplaceable subset goes offsite. A total
site loss means rebuilding the media library from source, which is acceptable.