# 0011 — Remote access: single Tailscale gateway, MGMT never advertised
**Status:** Accepted (Phase 3, 2026-07-13)
**Context:** Want remote access to lab services and the hypervisor with no inbound
exposure (threat-model goal). Where the tailnet terminates determines the blast radius of
a compromised tailnet device or key.
**Decision:** A single Tailscale gateway — phantom (VM 101, 10.0.30.10), subnet router for
10.0.30.0/24 plus exit node — is the only tailnet ingress. No tailscaled on executor. The
MGMT VLAN is not advertised; the hypervisor is reached via its VLAN 30 storage leg
(https://10.0.30.2:8006).
**Alternatives:** tailscaled on executor as a node (rejected — an agent on the
highest-value host, whose compromise is total); advertising 10.0.10.0/24 gated by ACLs
(rejected — policy control is weaker than structural absence).
**Consequences:** Remote access depends on phantom's health (accepted). Intra-VLAN-30
admin traffic is switch-local and invisible to tarkin — visibility comes instead from the
Tailscale admin console, phantom's auth/sshd logs, Proxmox access logs, and Pi-hole via
the tailnet DNS override. Tailnet ACLs remain default allow-all while the tailnet is
single-user (revisit: any non-owner device joins, or Phase 6 hardening). All cross-VLAN
and exit-node traffic funnels through one audited chokepoint (phantom → order66 → tarkin).
