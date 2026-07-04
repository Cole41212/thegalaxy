# 0009 — Infrastructure VM backups stay local (supersedes part of 0008)
**Status:** Accepted (Phase 2, incident-driven)
**Context:** 0008 moved all vzdump to the TrueNAS NAS. On 2026-06-23 the nightly backup of
tarkin deadlocked: the NFS path to the NAS routes through tarkin, the write stalled, and
tarkin's qemu froze in uninterruptible I/O — taking down routing and DHCP lab-wide.
**Decision:** VMs the backup path depends on (tarkin 100 — routing; archives 102 — the NAS
itself) back up only to local storage (ssd-vmstore); all other VMs continue to the NAS,
mounted soft (soft,timeo=30,retrans=3); the host gets a direct VLAN 30 interface
(vmbr1.30, 10.0.30.2/24) so host↔NAS traffic never traverses the firewall.
**Alternatives:** Keep everything on the NAS and accept the loop (unacceptable —
self-reinforcing outage); back everything up locally (wastes the pool).
**Consequences:** No backup path depends on its own subject; a NAS stall now degrades to
I/O errors instead of a frozen router.
