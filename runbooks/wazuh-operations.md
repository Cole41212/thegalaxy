# Runbook — Wazuh Operations (inquisitor)

Operating the SIEM, not building it. Design and sizing:
[ADR 0018](../decisions/0018-wazuh-deployment-model-and-sizing.md) ·
[ADR 0019](../decisions/0019-siem-data-placement-and-backup.md). Build log and the incidents
these procedures come from: [phases/phase-6-inquisitor-wazuh.md](../phases/phase-6-inquisitor-wazuh.md).

| | |
|---|---|
| Host | `inquisitor` — VM 105, 10.0.30.60, Wazuh 4.14.7 all-in-one |
| Services | `wazuh-indexer` · `wazuh-manager` · `wazuh-dashboard` · `filebeat` |
| Dashboard | `https://10.0.30.60` (self-signed) |
| Index data | `/var/lib/wazuh-indexer` on the 200 GB data disk, mounted by UUID |
| Manager data | `/var/ossec` on the system disk |
| Heap | `-Xms3g -Xmx3g` in `/etc/wazuh-indexer/jvm.options` — do not raise without raising VM RAM (ADR 0018 amendment) |
| Secrets | `/root/wazuh-artifacts/wazuh-install-files.tar` — mode 600 in a 700 directory. Never in this repo |

Run every command below **on inquisitor**. Check with `hostname` first — commands have been
run against the wrong host twice in this lab.

## Reading the admin password without leaving it in shell history

    read -s -p 'admin password: ' PW; echo

`read -s` takes the password from the keyboard without echoing it and without it ever
appearing on a command line. Use `"$PW"` in double quotes afterwards, and finish with:

    unset PW

To look a password up, read the artefact to the terminal instead of unpacking it to disk:

    sudo tar -O -xf /root/wazuh-artifacts/wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt

## Health checks that mean something

`systemctl is-active` reports that a unit is running, not that the service inside it works.
The manager ran for eleven days as `active` with its internal socket deleted. Check the thing
itself:

| Question | Command | Healthy answer |
|---|---|---|
| Are the units up? | `systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard filebeat` | four lines of `active` — necessary, not sufficient |
| Is the manager's internal database reachable? | `sudo ls -la /var/ossec/queue/db/wdb` | a line beginning `s` (a socket), with an mtime from the last manager start |
| Is the manager complaining? | `sudo grep -ciE "cannot find 'queue/db/wdb'\|already running" /var/ossec/logs/ossec.log` | no new matches since the last start |
| Is the index cluster healthy? | `curl -sk -u admin:"$PW" 'https://127.0.0.1:9200/_cluster/health?pretty'` | `"status" : "green"`, `"unassigned_shards" : 0` |
| Is the whole pipeline moving? | `curl -sk -u admin:"$PW" 'https://127.0.0.1:9200/wazuh-alerts-*/_count?pretty'` | a count that **rises** between two runs a few minutes apart |
| Is the index on the right disk? | `findmnt /var/lib/wazuh-indexer` | the data disk, not empty output |

The document count is the end-to-end test: it only rises if logcollector, analysisd, filebeat
and the indexer are all handing off correctly. Quote URLs that contain `?` or `&` — unquoted,
the shell acts on them before `curl` sees them.

## Symptom: the dashboard says "No API available to connect"

The index-pattern checks pass and the API check fails. The API process is usually fine; what
sits behind it is not. In order:

1. `sudo ss -tlnp | grep 55000` — is the API listening? If yes, the API is not the fault.
2. `sudo ls -la /var/ossec/queue/db/wdb` — does the socket exist? Missing means `wazuh-db`
   is unreachable, and every daemon that depends on it is failing quietly.
3. `sudo tail -30 /var/ossec/logs/ossec.log` — `Cannot find 'queue/db/wdb'` confirms it.

Recovery:

1. `sudo systemctl stop wazuh-manager`
2. `ps -eo pid,stat,lstart,args | grep -E '[w]azuh-|[o]ssec'` — **every** `/var/ossec/bin/`
   process must be gone. A daemon that survives the stop is the cause, not a detail.
3. `sudo systemctl start wazuh-manager`
4. Re-check the socket. Its mtime must be from this start.

If processes survive the stop, or show state `Z` (zombie), reboot the VM from the Proxmox GUI.
A zombie whose thread group still has live threads cannot be reaped by init, and
`systemctl stop` does not clear it.

Why this should not recur: the vendor unit ships `KillMode=process`, which lets a hung daemon
outlive a restart with its socket already unlinked. A drop-in at
`/etc/systemd/system/wazuh-manager.service.d/override.conf` sets `KillMode=control-group`, so
anything left after the graceful stop is killed with the cgroup.

## Patching

Unattended upgrades still install. What changed is that **nothing restarts Wazuh
automatically**: `/etc/needrestart/conf.d/99-wazuh.conf` excludes `wazuh-*` from needrestart,
because an unattended restart is what broke the manager on 2026-09-11.

The cost is that Wazuh keeps running against the old copy of an upgraded library until it is
restarted by hand. After library upgrades:

1. `sudo needrestart -b` — lists what is waiting for a restart.
2. Restart the affected Wazuh service deliberately, then run the health checks above.
3. A new kernel needs a VM reboot. Reboot from the Proxmox GUI, then confirm with `uname -r` —
   `/var/run/reboot-required` disappearing proves a reboot happened, not which kernel booted.

Upgrade the Wazuh packages themselves only on purpose, never as part of a general upgrade.

## Index lifecycle

- **Shards.** New alert indices get one primary shard from the order-10 template
  `wazuh-custom`. Do not edit Wazuh's own template (`wazuh`): filebeat rewrites it on every
  start and the edit disappears without an error.
- **Retention.** ISM policy `wazuh-retention-90d` deletes at 90 days. It attaches
  automatically to new `wazuh-alerts-4.x-*`, `wazuh-archives-4.x-*`, `wazuh-statistics-*` and
  `wazuh-monitoring-*` indices.
- **Never add `wazuh-states-*` to it.** Those are current-state indices with no date suffix,
  updated in place. Deleting one by age removes live inventory, not history.
- **A new index pattern needs one question answered first:** is each index a period of time,
  or the current state of something? Only the first kind can be aged out.

Coverage check — the total should match the number of time-series indices:

    curl -sk -u admin:"$PW" 'https://127.0.0.1:9200/_plugins/_ism/explain' | grep -o '"total_managed_indices":[0-9]*'

When attaching the policy by hand, `"failures": true` with "already has a policy" on every
index means the work is already done, not that it failed.

## Disk

Two volumes, two different failures:

- **`/var/lib/wazuh-indexer` (data disk).** OpenSearch stops allocating shards at 85% full,
  tries to move them away at 90%, and at 95% marks every index `read-only-allow-delete`. At
  that point the SIEM has stopped recording while every service still reads `active`.
  Retention exists to keep it away from that line.
- **`/` (system disk).** Holds `/var/ossec`. Vulnerability Detection is **off** and stays off
  while its feed would land here: enabled by default, it wrote 13.7 GB to this volume before
  any agent was enrolled. Filling root breaks journald, sshd and apt together.

Check both, separately:

    df -h /var/lib/wazuh-indexer /

If the index disk is ever not mounted, the indexer refuses to start (a `RequiresMountsFor`
drop-in), and the VM still boots (`nofail` in fstab). That is the intended failure: a reachable
box with a stopped service, instead of an index quietly filling the root volume.

## Verifying a change to the index

Ask the cluster, not the filesystem. Index files change continuously while the service runs,
so comparing directories on a live node reports hundreds of differences that mean nothing.
`_cluster/health` and `_cat/shards` are the evidence:

    curl -sk -u admin:"$PW" 'https://127.0.0.1:9200/_cat/shards?v'

Every shard `STARTED`, none `UNASSIGNED`.
