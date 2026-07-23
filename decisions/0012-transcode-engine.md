# 0012 — Transcode engine: RX 5700 passthrough to cantina
**Status:** Accepted (Phase 4, 2026-07-22)
**Context:** Phase 4 needs a hardware transcode engine for cantina (VM 106, Jellyfin). The
original route — sharing the iGPU's QuickSync into a VM via GVT-g (Intel's mediated iGPU
sharing) — is discontinued upstream and non-functional on PVE 9's kernel, eliminating
every shared-iGPU design. The lab's discrete card was identified as an XFX RX 5700 DD
Ultra 8 GB — Navi 10, non-XT confirmed via its 36 active CUs (compute units).
**Decision:** Pass the RX 5700 through whole to VM 106. The HD 630 iGPU stays with the
host as its console.
**Alternatives:** Full iGPU passthrough (rejected — takes the host console with it,
violating the host-console guardrail); GVT-g (rejected — dead upstream); Jellyfin in an
LXC sharing the host's /dev/dri (deferred — deviates from the VM-per-service pattern);
CPU-only interim (rejected — unnecessary once the card proved viable).
**Consequences:** Navi 10 has no FLR (Function Level Reset), so a VM stop→start can wedge
the card; recovery is a host reboot, with the vendor-reset kernel module as the documented
escalation if wedges become chronic. D3cold (the deepest PCI sleep state) is excluded via
udev as the config-space-wedge mitigation — which of host-reboot vs. D3cold exclusion
fixed the original reset assertion is recorded as ambiguous (falsifier: recurrence with
both mitigations in place). No AV1 (RDNA 1 predates it). HDR tone mapping unavailable —
Mesa's Rusticl (its OpenCL runtime) lacks VA-API↔OpenCL DMA-BUF interop; revisit if
Rusticl gains external-memory support or an HDR-capable library materializes. Measured
capacity: 4K HEVC Main10 → 1080p HEVC at 2.37× realtime (~82 fps), CPU idle.
