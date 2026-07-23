# Phase 4 — cantina (Jellyfin) & GPU Passthrough

**Goal:** Give the lab its media service and the hardware to run it: identify and
resurrect the discrete GPU, pass it through to a new VM — cantina (106, Jellyfin) — mount
the library read-only from archives, prove the transcode path end-to-end, and open the
low-trust VLANs exactly one port. Along the way: the transcode-engine decision (ADR 0012)
and the media ingestion path (ADR 0013).

**Status:** ✅ Complete

---

## Decisions & Rationale

### Transcode engine: RX 5700 passthrough (ADR 0012)

GVT-g — Intel's mediated iGPU sharing, the original route to QuickSync inside a VM — is
discontinued upstream and non-functional on PVE 9's kernel, which eliminated every
shared-iGPU design. The discrete card turned out to be an XFX RX 5700 DD Ultra 8 GB
(Navi 10; non-XT, confirmed by its 36 active CUs). Decision: pass it whole to VM 106,
HD 630 staying as host console. Full iGPU passthrough would violate the host-console
guardrail; Jellyfin-in-LXC sharing /dev/dri was deferred as a deviation from
VM-per-service; a CPU-only interim became pointless once the card proved viable. The
operational consequences — no FLR (a stop→start can wedge the card; recovery is host
reboot), no AV1, no HDR tone mapping on this stack — are recorded with their falsifiers
in ADR 0012, along with measured capacity: 4K HEVC Main10 → 1080p HEVC at 2.37× realtime,
CPU idle.

### Media ingestion: two doors, one dataset (ADR 0013)

Media must be writable from falcon (TRUSTED) and readable by cantina (SERVERS); one rw
path serving both would hand the internet-adjacent media server write access to the
library it serves. Instead: write door = authenticated SMB from TRUSTED as `coalish`
(dedicated non-admin user); read door = NFS mounted read-only on cantina. The dataset's
POSIX ACL carries named user `coalish` rwx + the mandatory mask rwx, with `Other` r-x
preserved — that r-x IS the NFS read path. A compromised cantina can stream the library
but cannot modify or encrypt it. Verified end-to-end (SMB write → NFS read of the same
file).

### The media mount: `ro,soft`, and deliberately looser timers than the backup guardrail

    ro,soft,timeo=150,retrans=3,noatime,_netdev,x-systemd.automount

- **ro** — the read door of ADR 0013, enforced at the mount as well as the export.
- **soft** — if archives stalls, reads return errors instead of hanging Jellyfin's
  scanners in uninterruptible sleep. Same failure philosophy as the nas-vmbackups
  guardrail (ADR 0009's deadlock lesson).
- **timeo=150,retrans=3** — looser than the guardrail's `timeo=30` on purpose: a backup
  writer must fail fast so a stalled NAS can't wedge vzdump, but a read-only library
  prefers riding out a 15-second hiccup over spraying scan errors. Different workload,
  different failure economics.
- **noatime** — no access-time bookkeeping for a media library.
- **_netdev,x-systemd.automount** — the mount materializes on first access; boot never
  blocks on the NAS, and a down NAS costs an error, not a boot hang.

### Native Jellyfin over Docker

Native install is the thinnest possible GPU plumbing — VA-API talks straight to /dev/dri
with one render-group membership, no container device mapping on top. It is apt-managed
(Jellyfin's own repo), and shipyard already demonstrates the container pattern —
repeating it here adds a debugging layer without adding portfolio signal. It also keeps
the VM-per-service shape that made the LXC option a deferral in ADR 0012.

### apt pinned to IPv4 on cantina

The lab is IPv4-only; apt burned time in IPv6 attempts (EAI_NODATA) before falling back.
ForceIPv4 ends that. The concurrent path-MTU finding (1400, from the ISP's double-NAT) is
environmental — documented, no action (log #5).

### HDR tone mapping: off, with explicit revisit triggers

Hardware tone mapping needs OpenCL running against the decoder's VA-API surfaces, and
Mesa's Rusticl (its OpenCL runtime) lacks VA-API↔OpenCL DMA-BUF interop — the zero-copy
handoff doesn't exist on this stack. Tone mapping is therefore off in Jellyfin rather
than left to fail per-stream. Revisit if Rusticl gains external-memory support or an
HDR-capable library materializes (ADR 0012). Until then the HDR path is untested — see
the verification caveat.

---

## Checklist

| Step | Status | Notes |
|------|--------|-------|
| GPU forensics — card identified & enumerating | ✅ | XFX RX 5700 DD Ultra 8 GB (Navi 10 non-XT, 36 CU); aux power was the blocker |
| ADR 0012 — transcode engine | ✅ | dGPU passthrough; GVT-g dead upstream |
| vfio prep on executor | ✅ | Early ID pin + softdeps, no-D3cold udev, initramfs rebuilt |
| cantina built (VM 106) + GPU attached | ✅ | Endpoints 03:00.0/.1 only; Primary GPU unchecked |
| Media mount from archives | ✅ | NFS `ro,soft` automount (rationale above) |
| Jellyfin installed (native) | ✅ | jellyfin-ffmpeg 7.1.4, bundled Mesa 25.0.7 |
| Hardware transcode proven | ✅ | 2.37× realtime / ~82 fps, zero-copy, CPU idle |
| Access policy | ✅ | :8096 pinholes (FAMILY/IOTGUEST) + ADR 0013 ingest path |
| Backup verified | ✅ | VM 106 in the nightly nas-vmbackups job |
| Docs | ✅ | ADRs 0012/0013 · runbook GPU addendum · network docs · inventory |

---

## Verification evidence

- **Pipeline fully hardware, zero-copy:** `-hwaccel vaapi -hwaccel_output_format vaapi`
  → `scale_vaapi` → `hevc_vaapi` — decode, scale, and encode all stay on the card;
  frames never round-trip through system RAM.
- **Throughput:** 4K HEVC Main10 → 1080p HEVC at **2.37× realtime (~82 fps)** on the
  200 Mbps jellyfish reference clip, CPU essentially idle throughout.
- **Caveat:** the clip is 10-bit **SDR** — this proves the Main10 decode path, not HDR.
  HDR tone mapping is untested and disabled (rationale above).
- **Toolchain identity:** jellyfin-ffmpeg 7.1.4 ships its own Mesa 25.0.7, distinct from
  the system's 25.2.8 — when debugging the transcode stack, debug the bundled one.
- **Backup:** VM 106's vzdump to nas-vmbackups verified (auto-enrolled via the job's
  "all except" selection, per ADR 0009).
- **Pinholes verified both directions from both VLANs:** allow — Jellyfin loads from
  FAMILY and IOTGUEST; deny — Portainer (10.0.30.25:9443) remains unreachable from both.

---

## Log

Recorded 2026-07-22 at phase close; entries in build order (per-entry dates weren't
tracked during the build).

### 1 — GPU absent from PCI enumeration

#### Issue
The discrete card didn't appear in `lspci` at all — carried in the inventory since
Phase 2 as "not yet enumerating."

#### Root cause
The PCIe aux power leads were unplugged. Without aux power a Navi 10 card doesn't respond
to configuration cycles at all — the slot's 75 W isn't enough to bring it up.

#### Fix
Connected the 8-pin + 6-pin leads; set BIOS to iGPU-primary + multi-monitor so the HD 630
keeps the host console while the dGPU still initializes.

#### Learning
Absent from `lspci` means physical/firmware — power, seating, BIOS — never software. No
driver setting can fix a card the bus can't see.

### 2 — Host offline after the GPU began enumerating

#### Issue
First reboot with the card alive: executor unreachable on every interface.

#### Root cause
The card's on-die PCIe switch consumed bus numbers and renumbered the bus, renaming the
LAN NIC `enp3s0` → `enp5s0` and orphaning the bridge config. (The docs had said `enp1s0`
— stale on top of stale — and an unused `enp1s0` stanza was still sitting in
`/etc/network/interfaces`.)

#### Fix
`bridge-ports` updated to `enp5s0`; stale `enp1s0` stanza removed. The same sweep
corrected archives' hostpci entries 05/06 → 07/08 (the ASM1064 and SAS3008 shifted too).

#### Learning
Predictable NIC names and hostpci addresses BOTH encode PCI topology, and BOTH move when
it changes — re-verify both after any hardware change.

### 3 — cantina wouldn't start: failed reset + `pci_irq_handler` assertion

#### Issue
First VM start with the GPU attached died on a failed device reset with a kernel
`pci_irq_handler` assertion.

#### Root cause
Navi 10 has no FLR, and the card's config space was wedged — by amdgpu residue from the
host boot, by D3cold runtime power management, or both.

#### Fix
Early vfio-pci ID bind (amdgpu never touches the card again), the no-D3cold udev rule,
and a fresh host boot. VM started clean.

#### Learning
Both mitigations retained; which one actually cleared it is recorded as ambiguous rather
than guessed. Falsifier: a recurrence with both in place reopens the diagnosis (then:
vendor-reset, per ADR 0012).

### 4 — IOMMU group 2 warning: four functions, one group

#### Issue
Proxmox warned that the GPU shares IOMMU group 2 with three other functions — normally a
passthrough stopper.

#### Root cause
Not an isolation failure: the card carries an on-die PCIe switch, so upstream port,
downstream port, GPU, and HDMI audio are four functions of one physical card in one group.

#### Fix
Pass the endpoints (03:00.0 + 03:00.1); the bridge functions stay bound to `pcieport` on
the host.

#### Learning
Bridges are host infrastructure — vfio's group-viability check exempts them because they
do no DMA of their own, so endpoints-only is safe and correct here. The ACS override
patch stays forbidden: it fakes isolation instead of understanding the topology.

### 5 — apt at 416 B/s with EAI_NODATA

#### Issue
On cantina, apt crawled at 416 B/s with EAI_NODATA failures — while tunnel traffic
(Tailscale) stayed perfectly healthy.

#### Root cause
Two concurrent faults. Primary: apt attempting IPv6 on an IPv4-only lab (EAI_NODATA is
getaddrinfo's "no address for the requested family"). Secondary: path MTU 1400 from the
ISP's double-NAT — environmental, not lab-fixable.

#### Fix
apt pinned to IPv4 (ForceIPv4). The MTU finding is documented, no action.

#### Learning
Two faults at once means the first fix "not working" doesn't falsify it — the workload
test discriminates. Tailscale-fine-while-HTTP-crawls is the small/large-packet split that
fingerprints MTU trouble: Tailscale clamps its own packet size and never hits the 1400
ceiling, full-size HTTP segments do.

### 6 — cantina missing from Kea's lease table

#### Issue
cantina never appeared in Kea's leases, so there was nothing for the naming machinery to
register.

#### Root cause
The OS was configured static-in-OS, and a static host never speaks DHCP — a Kea
reservation without a DHCP exchange is a row of config, not a lease.

#### Fix
netplan `dhcp4: true` plus a Kea reservation pinning 10.0.30.50 (`bc:24:11:78:00:83`).
The lease now exists and carries the name.

#### Learning
Reservation ≠ lease. The lab's naming rides the DHCP exchange, so hosts that should have
names must actually DHCP — a deliberate deviation from the static-in-OS pattern of the
other VLAN 30 VMs (recorded in network/ip-scheme.md).

### 7 — cantina not resolving despite reservation + Unbound restart

#### Issue
Lease present, reservation present, Unbound restarted — `cantina` still didn't resolve.

#### Root cause
Unbound's registration checkboxes (General → "Register ISC DHCP4 Leases" and "Register
DHCP Static Mappings") were off. With them off, reservation + restart registers nothing.

#### Fix
Enabled both; the name appeared.

#### Learning
The naming design had an undocumented dependency — now a documented guardrail in
network/dns-design.md. Also a wrong prediction worth recording: the ISC-named options
looked like dead ISC-era toggles that couldn't work with Kea — tested, they consume Kea's
lease data on OPNsense 26.x. Corrected and documented rather than trusted.

### 8 — Kea's SERVERS DNS-server order kept reverting

#### Issue
Editing the SERVERS subnet's DNS servers in place, the saved list came back re-sorted —
wrong resolver first.

#### Root cause
Kea UI quirk: ordered lists can silently re-sort on an in-place edit.

#### Fix
Remove all entries → re-add in the intended order → Apply. Confirmed the delivered order
on a client with `resolvectl status` after a renew.

#### Learning
Verify on the wire, not in the UI — the config page shows intent; the lease shows truth.

### 9 — TrueNAS ACL save failed: `Error: dacl`

#### Issue
Saving the media dataset's ACL (named user `coalish` rwx) failed with a bare
`Error: dacl`.

#### Root cause
POSIX ACLs require a mask entry once any named entry exists; the submitted ACL had none.

#### Fix
Added mask rwx alongside the named entry; `Other` stayed r-x.

#### Learning
The mask is a ceiling over named-user/group entries — and `Other` sits outside it, which
is exactly why the NFS read path (`Other` r-x) survives ingest-permission changes.

### 10 — The media automount never fired

#### Issue
fstab entry with `x-systemd.automount` in place, but /mnt/media stayed empty and
`systemctl list-units` showed no automount unit at all.

#### Root cause
Two behaviors stacked: `daemon-reload` loads generated units but does not start them, and
`list-units` hides inactive units by default — so the unit existed, inactive and
invisible.

#### Fix
`systemctl start mnt-media.automount` — armed the automount; first access mounted.

#### Learning
Reload ≠ start: a freshly generated unit must be started once (boot handles it
thereafter). And `list-units` without `--all` is a filter, not an inventory.

### 11 — Minimized install: no nano, ping, dig, or vainfo

#### Issue
Basic tools missing mid-troubleshoot — editor, reachability, DNS, and VA-API probes all
absent.

#### Root cause
Ubuntu minimized install — diagnostic tooling stripped by design.

#### Fix
Installed what the box should keep; meanwhile leaned on what's always present: bash's
`/dev/tcp` for reachability, `resolvectl query` and `getent hosts` for DNS.

#### Learning
Know the zero-install fallbacks before needing them — a minimized image turns "trivial
check" into "install a package first" at exactly the moment apt itself might be the
broken thing (#5).

### 12 — Every GPU counter read 0 during a proven transcode

#### Issue
While ffmpeg demonstrably transcoded on the card, `gpu_busy_percent` read 0 and radeontop
showed nothing moving.

#### Root cause
Three measurement artifacts stacked: `gpu_busy_percent` covers the GFX (3D) block only —
VCN (Video Core Next, the encode/decode engine doing the actual work) isn't in it;
radeontop has no VCN row on Navi 10 at all; and at 2.37× realtime the whole clip was done
in ~13 s, so the sampling window was nearly gone before watching began.

#### Fix
Judged the transcode by decisive evidence instead: the encoder line in the ffmpeg log
(`hevc_vaapi`) plus an idle CPU during the run.

#### Learning
Know what a counter measures before trusting its zero. The encoder line in the log is
this domain's reply code — the same lesson as Phase 3's DNS incident: verify the
mechanism, not the vibe.

---

## Deferred / follow-ups

- **vendor-reset module** — only if stop→start wedges recur with both mitigations in
  place (the ADR 0012 falsifier firing).
- **HDR tone mapping** — revisit on Rusticl external-memory support or an HDR-capable
  library (ADR 0012).
- **Path MTU 1400** — environmental (ISP double-NAT); documented, no action.
- **SMB ingest scope** — the share is currently media-dataset-wide; fine for one user,
  revisit per-user scoping if users multiply (ADR 0013).
- **Library organization** — Movies/Shows structure as content grows.
