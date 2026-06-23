# 0008 — SSD #1 repurposed to VM-disk store; vzdump to NAS
**Status:** Accepted (post-Phase 2)
**Context:** 465 GB NVMe is tight for ~10 VMs; vzdump was targeting SSD #1, duplicating
capacity the NAS already has.
**Decision:** SSD #1 → additional VM-disk store (`ssd-vmstore`, content type Disk image);
vzdump target → TrueNAS `vm-backups` (NFS); SSD #2 stays Wazuh data.
**Alternatives:** Keep SSD #1 as backup target (wastes it; NAS has the space); PBS VM (no
room on the NVMe).
**Consequences:** More VM headroom. archives must not back up to itself; its boot/config
goes to NVMe/SSD (see 0005).