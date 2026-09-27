# 0014 — Photo platform: Immich; general file sync deferred
**Status:** Accepted (Phase 5, 2026-07-24) — revisit trigger (b) evaluated and declined
2026-08-17. See the amendment below.
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

---

## Amendment — 2026-08-17 (revisit trigger (b): evaluated, declined)

Trigger **(b)** — "Phase 6 wanting an auth-log-rich application target" — fired at Phase 6
planning. It was evaluated and **declined**. The decision text above stands unchanged; general
file sync remains deferred and `holocron/files` remains empty.

**The requirement is already satisfied by services that exist.** vault (Immich), cantina
(Jellyfin), shipyard (Portainer and Crafty), and order66 (Pi-hole) each present a web login and
an admin surface, and all four are agent hosts under
[ADR 0020](0020-agent-coverage-and-log-sources.md). That is four independent sources of
application authentication events, across four different codebases, with genuine day-to-day
traffic behind them. Nextcloud would be a fifth of the same kind, not a new kind.

**Standing up a service in order to have something to detect against inverts the priority.** The
SIEM exists to observe the lab; the lab does not exist to feed the SIEM. Deploying a file-sync
platform for telemetry reasons would add a service, an attack surface, a patch obligation, and
RAM the host does not have — ADR 0018 measures roughly 15 GiB genuinely available, of which
inquisitor takes 8 — in exchange for log lines already available from services that are
carrying real work. If a detection needs exercising, Phase 7's attack range generates events
deliberately, which is the correct instrument for that job.

**Triggers (a) and (c) remain unfired.** External share links are still not needed — sharing to
non-household users is a Phase 8 question tied to the domain and Cloudflare Tunnel (ADR 0017) —
and SMB plus the tailnet has not yet proven insufficient for remote file access. If either
fires, this ADR reopens on its own merits rather than on the SIEM's.

**Closes Phase 5 deferred item 5.**
