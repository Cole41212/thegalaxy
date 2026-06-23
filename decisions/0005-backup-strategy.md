# 0005 — 3-2-1 backups with a Tailscale-only offsite Pi
**Status:** Accepted (Phase 3+)
**Context:** RAIDZ2 is redundancy, not backup; everything currently lives on one host.
**Decision:** vzdump → TrueNAS (local convenience); irreplaceable subset (files, configs,
key media) → Pi 5 + 6 TB at a relative's, reachable only over Tailscale; configs versioned
in git.
**Alternatives:** Cloud backup (cost/egress for ~TBs); WD MyBook Live (EOL/CVE — rejected);
no offsite (fails 3-2-1).
**Consequences:** Survives site loss for what matters. Bulk media is single-site (accepted).