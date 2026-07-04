# Runbook — Storage Deadlock Recovery (NFS backup wedging a VM)

## Symptoms
- Backup jobs fail with "could not activate storage '<name>'".
- A VM's console is dead/unresponsive; operations on it hang.
- If that VM is tarkin: DHCP stops serving and inter-VLAN routing is down lab-wide.
- Host kernel log: `nfs: server <ip> not responding, timed out`.

## Cause pattern
vzdump writing over NFS whose network path depends on the VM being backed up. The NFS
write stalls → qemu enters uninterruptible I/O sleep (D state) — unkillable until the
blocked I/O is released. If the VM is the router, the stall also takes down the path NFS
needs to recover: a self-reinforcing deadlock that cannot self-recover.

## Recovery (in order)
1. **Datacenter → Storage → disable the NFS storage entry** — stops pvestatd from
   re-probing the dead mount.
2. **Host shell:** `umount -f -l /mnt/pve/<storage>` — CLI required; no GUI can
   force-detach a hung NFS mount. `-f` forces, `-l` lazily detaches despite stuck I/O.
   This releases the blocked I/O holding qemu.
3. **Hard-stop the wedged VM** (Stop, or `qm stop <id>`), then start it again.
4. **Verify DHCP and routing** — client pulls a lease, inter-VLAN traffic flows.
5. Later: delete the partial `.vma.zst` from the backup target.

## Prevention
- Infrastructure VMs back up locally only (`ssd-vmstore`) — the backup path must never
  depend on the VM being backed up (decisions/0009).
- NFS backup storage mounted with `soft,timeo=30,retrans=3` — a stall returns I/O errors
  instead of freezing qemu.
- The host reaches the NAS via its own VLAN 30 leg (`vmbr1.30`) so storage traffic never
  traverses the firewall.
