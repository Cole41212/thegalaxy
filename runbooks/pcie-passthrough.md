# Runbook — PCIe Passthrough (Proxmox 9)

Passing whole PCI devices to VMs on executor: storage controllers → archives
(TrueNAS/ZFS gets direct hardware access), and the RX 5700 → cantina (Jellyfin
transcode — see the GPU addendum at the end).

## 1. Verify IOMMU (CLI — no GUI equivalent)
    dmesg | grep -e DMAR -e IOMMU
Expect "Intel(R) Virtualization Technology for Directed I/O". On PVE 9 (kernel 6.x)
Intel VT-d is on by default. If absent: enable VT-d in BIOS; only if still absent, add
`intel_iommu=on iommu=pt` to the kernel cmdline and reboot.

No manual VFIO module loading or driver blacklisting is needed on PVE 9 — adding the
device in the GUI binds vfio-pci automatically. (True for storage controllers; GPUs are
the exception — see the GPU addendum.)

## 2. Identify controllers + verify IOMMU isolation (CLI — Proxmox API)
    pvesh get /nodes/pve/hardware/pci --pci-class-blacklist ""
Find the controllers by class: SATA = 0x0106xx, SAS = 0x0107xx. Note the `iommugroup`.
Each controller you pass through must be in its own group, clear of the NVMe and the
motherboard SATA controller (which keeps the SSDs). If a controller shares a group with
something you need, move the card to a different PCIe slot.

executor result (post-GPU renumbering, Phase 4): ASM1064 SATA = 07:00.0 (group 15);
SAS3008 = 08:00.0 (group 16) — both isolated. They sat at 05:00.0/06:00.0 until the GPU
began enumerating; the bus renumbered and archives' hostpci entries had to be corrected
to match. Re-verify addresses after any hardware change.

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

## GPU passthrough addendum (RX 5700 → cantina, Phase 4)

The automatic GUI-time vfio-pci bind (section 1) is enough for HBAs, whose host drivers
release cleanly. GPUs are different: a live graphics driver doesn't release the card
cleanly, so GPUs get an early vfio-pci ID pin in the initramfs — the card must never be
claimed by amdgpu in the first place.

### Early ID pin (CLI — no GUI path for modprobe/initramfs config)
`/etc/modprobe.d/vfio.conf` — all four of the card's IDs, plus softdeps so the competing
drivers load only after vfio-pci:

    options vfio-pci ids=1002:1478,1002:1479,1002:731f,1002:ab38
    softdep amdgpu pre: vfio-pci
    softdep radeon pre: vfio-pci
    softdep snd_hda_intel pre: vfio-pci
    softdep pcieport pre: vfio-pci

Then `update-initramfs -u -k all` and reboot.

### Bridge functions legitimately stay `pcieport`
After the pin, 01:00.0 / 02:00.0 (the card's on-die PCIe switch) still bind `pcieport`
even though their IDs are in the vfio list. Correct and expected: vfio's group-viability
check exempts bridge infrastructure (bridges do no DMA of their own), so passing only the
endpoints out of IOMMU group 2 is safe. Never "fix" the shared-group warning with the ACS
override patch — it lies to the IOMMU about isolation, and nothing here needs splitting.

### D3cold exclusion
`/etc/udev/rules.d/99-navi-no-d3cold.rules` clears `d3cold_allowed` for the card's
functions. Navi 10 can wedge its config space around D3cold (the deepest PCI sleep
state), and with no FLR that wedge survives until host reboot. One of the paired
mitigations recorded in ADR 0012 (attribution ambiguous; falsifier documented there).

### VM config (GUI)
- Add the endpoints only: 03:00.0 with **All Functions** (brings the 03:00.1 audio
  function along) + **PCI-Express**. The bridge functions are never added.
- **Primary GPU stays unchecked** — the emulated display remains primary, keeping the
  noVNC console usable while VA-API uses the card.

### No-FLR operational note
Navi 10 has no Function Level Reset. If a cantina (VM 106) stop→start fails with a
failed reset / `pci_irq_handler` assertion, recover with a host reboot — nothing softer
reliably clears the wedge. If wedges become chronic, the documented escalation is the
vendor-reset module (ADR 0012).