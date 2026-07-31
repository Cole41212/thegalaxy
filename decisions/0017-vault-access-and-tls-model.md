# 0017 — Access and TLS model for vault
**Status:** Accepted (Phase 5, 2026-07-24)
**Context:** The lab has no domain yet — Phase 8 acquires one. Immich's FAQ is explicit that
self-signed certificates break mobile video playback and upload, and that the recommended
alternatives are a VPN or a real certificate.
**Decision:** Plain HTTP on `http://10.0.30.40:2283`, one URL for LAN and tailnet alike — no
URL-switching, which has multiple open upstream bugs. Remote access rides the existing
phantom subnet route; no new tailnet node, so ADR 0011's single-ingress property stays
intact. No new firewall rules: a single user on TRUSTED, and no FAMILY pinhole.
**External service egress (Immich Server Privacy):** three outbound-egress consent
decisions, distinct from the no-inbound-exposure posture. Version Check
(version.immich.cloud) — ENABLED: it sends only the running version string and returns
update-available notices, which is what serves the higher-patch-cadence consequence of ADR
0014; trivial privacy cost, real security value. Map tiles (tiles.immich.cloud) — ENABLED:
tiles load on demand when the map view opens, so the tile CDN can infer approximate photo
geography from the regions requested; no photos and no exact coordinates leave the box, and
place names come from the local reverse-geocoder — accepted for a single-user personal
library. Google Cast — DISABLED: it pulls external Google code into the web client for no
stated need, the primary client being the iOS app; a broader dependency than warranted.
**Alternatives:** Self-signed TLS (rejected — breaks the mobile client, per upstream); a
tailnet node on vault itself (rejected — a second tailnet ingress, against ADR 0011);
separate LAN and remote URLs (rejected — open upstream URL-switching bugs).
**Consequences:** Accepted, time-boxed risk: cleartext on VLAN 30 and TRUSTED WiFi. The
positioned adversary in the threat model — a compromised IoT or guest device — is contained
to VLAN 50 with no lab reach. Remediation is scheduled for Phase 8: Caddy with ACME DNS-01
on the portfolio domain, which needs zero inbound ports. Sharing to non-household users is
also Phase 8; Cloudflare Tunnel's ~100 MB upload cap means the tunnel is for shared-link
viewing only — upload and backup stay on the tailnet permanently.
