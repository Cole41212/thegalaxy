# 0003 — TrueNAS as a VM with whole-controller passthrough
**Status:** Accepted (Phase 2)
**Context:** ZFS needs direct hardware access (cache flush, SMART, firmware) for integrity;
12 HDDs span two add-in controllers.
**Decision:** Pass the ASM1064 SATA (grp 15) and SAS3008 HBA (grp 16) controllers whole to
archives; SSDs/NVMe stay on the motherboard controller for Proxmox.
**Alternatives:** Bare-metal TrueNAS (loses consolidation); Proxmox-native ZFS (no TrueNAS
tooling); per-disk passthrough (breaks ZFS guarantees).
**Consequences:** Near-bare-metal ZFS in a VM. Requires verified IOMMU isolation; passed
controllers' drives vanish from the host while the VM runs.