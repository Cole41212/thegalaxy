# Runbook — Offsite Replication (echo-base / carbonite)

Operational since 2026-08-14. Design and rationale: [ADR 0010](../decisions/0010-offsite-replication-mechanism.md).
Build log: [phases/phase-5-vault-immich.md](../phases/phase-5-vault-immich.md).

## What this is

`echo-base` (Raspberry Pi 5, Ubuntu 24.04 arm64) at mom's house **pulls** ZFS snapshots from
`archives` over Tailscale, nightly, into an encrypted single-disk pool called `carbonite`.

The direction matters. The lab holds no credential that can touch the offsite copy, and the
Pi's credential into the lab is read-only. Nothing in the lab can encrypt, delete, or corrupt
what is offsite — that is the whole point of the design, so **do not "fix" a problem by
granting `pibackup` more ZFS permissions.**

| | |
|---|---|
| Appliance | `echo-base` — DHCP on mom's LAN, reached only over the tailnet |
| Pool | `carbonite` — single 6 TB disk, `aes-256-gcm`, key at `/root/.carbonite.key` (mode 400) |
| Datasets | `carbonite/photos` ← `holocron/photos` · `carbonite/media` ← `holocron/media` |
| Source account | `pibackup` on archives (UID 3002) — read-only ZFS delegation |
| Timers | `syncoid-media` 04:00 daily · `syncoid-photos` 04:30 daily (both `Persistent=true`) |
| Monitoring | healthchecks.io dead-man's switch, one check per dataset, pinged on success only |

## Check status

Run on echo-base.

```bash
systemctl list-timers 'syncoid-*'
```

Shows last fire and next fire. If `NEXT` is blank the timer is not enabled.

```bash
journalctl -u syncoid-photos.service -n 50 --no-pager
```

Swap in `syncoid-media` for the other dataset. Add `-f` to follow a run live.

```bash
zfs list -t snapshot -o name,used,creation carbonite/photos | tail -5
```

The newest snapshot should be from the most recent nightly run. Because replication uses
`--no-sync-snap`, it mirrors **TrueNAS periodic snapshots** — so the newest snapshot offsite
tracks the newest periodic snapshot on the source, not live state. A backup that is one
snapshot interval behind is working as designed.

```bash
zpool status carbonite
zfs get encryption,keystatus carbonite/photos
```

Expect `ONLINE` with no errors, `aes-256-gcm`, and `keystatus available`.

## Run a pull manually

```bash
sudo syncoid --no-privilege-elevation --sshkey=/root/.ssh/id_ed25519 --no-sync-snap pibackup@10.0.30.20:holocron/photos carbonite/photos
```

Substitute `media` for `photos` as needed. The three flags are all load-bearing:

- `--no-privilege-elevation` — syncoid otherwise tries `sudo` on the source. `pibackup` has
  no sudo by design and needs none; the `zfs allow` delegation covers exactly what it does.
- `--sshkey=` — the job runs as root, and root has its own `~/.ssh`. Without this it will not
  find a usable key.
- `--no-sync-snap` — stops syncoid creating its own sync snapshot, which it would then need
  `destroy` to clean up. `pibackup` has no `destroy` verb, deliberately.

## Restore a file from a snapshot

Snapshot directories automount on access and require root to traverse.

```bash
sudo -i
```

Then, in that root shell:

```bash
ls /carbonite/photos/.zfs/snapshot/
```

Pick a snapshot and copy what you need out of it:

```bash
cp /carbonite/photos/.zfs/snapshot/<snap>/library/admin/<year>/<file> /tmp/
```

`sudo cd` does not work — `cd` is a shell builtin, so `sudo cd /carbonite/...` silently
changes directory in a subshell that exits immediately. Use `sudo -i` first, then `cd`.

To verify a restored file against the source, compare hashes rather than sizes:

```bash
sha256sum /carbonite/photos/library/admin/<year>/<file>
```

and run the same on archives against `/mnt/holocron/photos/...`. This is the check that
satisfied the ADR 0016 gate — "a copy exists" is not the same claim as "data is recoverable."

## Restoring Immich specifically

`carbonite/photos` contains all six Immich directories, including `backups`. Per ADR 0016 a
restore is the logical database dump **plus** `UPLOAD_LOCATION`, together — never the dump
alone. Both halves are in this one dataset, which is why it is a single replication stream.

## When a healthcheck alert fires

An alert means a run did **not** report success. The ping is sent from `ExecStartPost`, so it
only fires when `ExecStart` succeeded — a failed run produces silence, and the silence is the
alert. Work outward from the cheapest test:

1. **Is echo-base up and on the tailnet?** From the lab: `tailscale status`. A Pi that lost
   power comes back on its own; `Persistent=true` catches up the missed run.
2. **Can it reach archives?** On echo-base: `ping 10.0.30.20` — expect `ttl=63` (one hop via
   phantom). If this fails, check `tailscale status` on echo-base and confirm it still
   accepts phantom's advertised route (`tailscale up --accept-routes`). Node key expiry is
   disabled on this node, but confirm it has not been re-enabled.
3. **Does SSH work non-interactively?** This is the form the job actually uses:
   ```bash
   sudo ssh -i /root/.ssh/id_ed25519 pibackup@10.0.30.20 "which zfs"
   ```
   Expect `/usr/sbin/zfs`. An interactive login proves nothing about a non-interactive one.
4. **Read the journal:** `journalctl -u syncoid-photos.service -n 100 --no-pager`.
5. **Is the pool healthy and unlocked?** `zpool status carbonite` and
   `zfs get keystatus carbonite`.

### Known failure modes

- **`bash: zfs: command not found`** — TrueNAS non-interactive shells do not source
  `/etc/profile`, and the ZFS binaries live in `/usr/sbin`. The fix is a PATH entry in
  `/etc/environment` on archives. **`/etc/environment` edits may not survive a TrueNAS
  update — check this first if replication breaks right after an upgrade.** Adding PATH to
  `.bashrc` does not work; bash does not source it for non-interactive sessions here.
- **`cannot destroy snapshots: permission denied`** — means `--no-sync-snap` was omitted.
  Add the flag. Do **not** grant `pibackup` the `destroy` verb.
- **`Permission denied (publickey)` under sudo but not interactively** — root has its own
  `~/.ssh` and cannot see the user's key. Use `--sshkey=/root/.ssh/id_ed25519`.
- **Datasets locked after a reboot** — Ubuntu does not auto-load a ZFS keyfile at boot. The
  `zfs-load-key-carbonite.service` unit handles this (oneshot,
  `After=zfs-import.target`, `Before=zfs-mount.service`). Check it with
  `systemctl status zfs-load-key-carbonite`.
- **Replication succeeds but the data is a day stale** — check timezones on *both* machines.
  Both must be `America/New_York`; systemd `OnCalendar` is interpreted in system-local time,
  so a clock mismatch makes the timer fire before the night's snapshot exists. This produces
  no error at all.

## Escalation: the falsifier

ADR 0010 keeps restic-to-an-append-only-rest-server as the fallback if ZFS-on-USB proves
unreliable. The seed showed zero USB resets, re-enumerations, or SCSI errors across a 154 GB
sustained write, and the enclosure negotiated UAS rather than falling back to BOT. If that
changes — repeated USB resets in `dmesg`, pool suspensions, or checksum errors that are not
explained by a failing disk — that is the falsifier firing, and the fallback becomes live
again rather than theoretical.

Because `carbonite` is a single disk, a checksum error offsite cannot self-heal. The recovery
is to re-replicate the affected dataset from `holocron`, which is intact and RAIDZ2-protected.
This is an accepted property of the design, not a defect.
