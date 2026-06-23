# Runbook — PCIe Controller Passthrough (Proxmox 9 → TrueNAS)

Passing a whole storage controller to a VM so the guest (TrueNAS/ZFS) gets direct
hardware access. Used for archives (TrueNAS) on executor.

## 1. Verify IOMMU (CLI — no GUI equivalent)
    dmesg | grep -e DMAR -e IOMMU
Expect "Intel(R) Virtualization Technology for Directed I/O". On PVE 9 (kernel 6.x)
Intel VT-d is on by default. If absent: enable VT-d in BIOS; only if still absent, add
`intel_iommu=on iommu=pt` to the kernel cmdline and reboot.

No manual VFIO module loading or driver blacklisting is needed on PVE 9 — adding the
device in the GUI binds vfio-pci automatically.

## 2. Identify controllers + verify IOMMU isolation (CLI — Proxmox API)
    pvesh get /nodes/pve/hardware/pci --pci-class-blacklist ""
Find the controllers by class: SATA = 0x0106xx, SAS = 0x0107xx. Note the `iommugroup`.
Each controller you pass through must be in its own group, clear of the NVMe and the
motherboard SATA controller (which keeps the SSDs). If a controller shares a group with
something you need, move the card to a different PCIe slot.

executor result: ASM1064 SATA = 05:00.0 (group 15); SAS3008 = 06:00.0 (group 16) — both
isolated.

## 3. Create the VM (GUI)
- Machine: q35 (required for PCIe passthrough)
- BIOS: OVMF (UEFI) + EFI disk — **uncheck "Pre-Enroll keys"** so Secure Boot is OFF
  (TrueNAS doesn't support Secure Boot; pre-enrolled keys cause "Access Denied" at boot)
- CPU: host; RAM: ballooning OFF (ZFS sizes ARC to RAM)
- Boot disk: small SCSI disk on local-lvm (install target — NOT the pool drives)

## 4. Add the controller(s) (GUI)
VM → Hardware → Add → PCI Device → Raw Device → select the controller →
check **All Functions** + **PCI-Express** → Add. Repeat per controller.

## Gotchas
- After the VM starts, the passed-through controllers' drives **disappear from the host**
  Disks view and stay hidden even after VM shutdown. This is normal (the host handed the
  controller to the VM), not a fault.
- noVNC console showing colored static during install = cosmetic mode-switch glitch;
  reopen the console or change Display type. The OS is running underneath.
- SAS3008 has two SFF-8643 ports, 4 drives each — both cables must be seated for >4 drives.
- A drive that spins but doesn't appear = data-cable problem, not power. Reseat its cable.
- ASM1064 is natively 4 SATA ports; avoid any port-multiplier ports on >4-port cards.