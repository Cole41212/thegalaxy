# 0014 — Photo platform: Immich; general file sync deferred
**Status:** Accepted (Phase 5, 2026-07-24)
**Context:** Phase 5 was scoped as Nextcloud (general files). The real requirement is
replacing a paid iCloud+ subscription — ~145 GB of owned originals, target ~17k assets —
with self-hosted photo management: mobile auto-backup, timeline, albums, face recognition,
semantic search, and text-in-image (OCR). Nextcloud approximates all of that through
add-ons; Immich is purpose-built for it.
**Decision:** VM 107 (vault, 10.0.30.40) runs Immich, pinned to v3.0.3. General file
sync/share is deferred; revisit at Phase 6 planning.
**Alternatives:** Nextcloud + Memories/Recognize (rejected — a bolt-on photo stack over a
file-sync core, with weaker mobile backup); both at once (rejected — scope, plus two
applications owning the same photos); PhotoPrism (rejected — less mature mobile clients and
background backup).
**Consequences:** No web file UI and no share links until the revisit; file access stays SMB
(ADR 0013) plus NFS. Immich does not backport patches, so this box carries a higher patch
cadence than the rest of the lab — a Phase 6 monitoring input. Revisit triggers: (a) a need
for external share links, (b) Phase 6 wanting an auth-log-rich application target, (c) SMB
proving insufficient for remote file access.
