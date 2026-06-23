# 0006 — Public services via Cloudflare Tunnel + a DMZ VLAN
**Status:** Accepted (planned, Phase 8)
**Context:** Want to self-host the portfolio site without exposing home IP or opening ports.
**Decision:** Run cloudflared on the web host (outbound-only tunnel); place internet-facing
services on a dedicated DMZ VLAN isolated from the rest of the lab. Optional GitHub Pages
mirror for always-up reliability.
**Alternatives:** Port-forward + DDNS (exposes IP/ports); reverse proxy on WAN (still
exposed); pure GitHub Pages (no self-host learning).
**Consequences:** Zero inbound exposure, free TLS/WAF/DDoS. Adds a DMZ VLAN + firewall rules;
depends on Cloudflare.