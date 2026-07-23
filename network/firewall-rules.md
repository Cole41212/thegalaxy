# Firewall Rules (OPNsense / tarkin — Rules [new])

Top-down evaluation, first match wins. Order within an interface matters.

## TRUSTED (VLAN 20)
| Action | Source | Destination | Port | Notes |
|---|---|---|---|---|
| pass | TRUSTED net | any | any | Full internet + homelab |

## SERVERS (VLAN 30)
| pass/block | Source | Destination | Port | Notes |
|---|---|---|---|---|
| block | SERVERS net | TRUSTED net | any | No path back to client devices |
| pass | SERVERS net | any | any | Internet allowed |

## FAMILY (VLAN 40)
| Action | Source | Destination | Port | Notes |
|---|---|---|---|---|
| pass | FAMILY net | 10.0.30.53 | 53 (TCP/UDP) | DNS to order66 (Pi-hole) — must sit above block |
| pass | FAMILY net | 10.0.30.1 | 53 (TCP/UDP) | DNS fallback to tarkin (Unbound) — above block |
| pass | FAMILY net | 10.0.30.50 | 8096 (TCP) | Jellyfin on cantina — above block (Phase 4) |
| block | FAMILY net | 10.0.0.0/16 | any | No homelab access |
| pass | FAMILY net | any | any | Internet only |

## IOTGUEST (VLAN 50)
| Action | Source | Destination | Port | Notes |
|---|---|---|---|---|
| pass | IOTGUEST net | 10.0.30.53 | 53 (TCP/UDP) | DNS to order66 (Pi-hole) — must sit above block |
| pass | IOTGUEST net | 10.0.30.1 | 53 (TCP/UDP) | DNS fallback to tarkin (Unbound) — above block |
| pass | IOTGUEST net | 10.0.30.50 | 8096 (TCP) | Jellyfin on cantina — above block (Phase 4) |
| block | IOTGUEST net | 10.0.0.0/16 | any | No homelab access |
| pass | IOTGUEST net | any | any | Internet only |

## SECLAB (VLAN 60)
| Action | Source | Destination | Port | Notes |
|---|---|---|---|---|
| block | SECLAB net | 10.0.0.0/16 | any | Fully isolated from homelab |
| pass | SECLAB net | any | any | Internet (to be tightened in Phase 7 — rogue gets none) |

SECLAB has no port-53 exception, so the DHCP-provided DNS (10.0.60.1) is unreachable —
the VLAN has no working DNS by default. Deliberate: nothing lives there yet, and Phase 7
designs SECLAB's DNS posture on purpose (see network/dns-design.md).

## MGMT (VLAN 10)
| block | MGMT net | TRUSTED net | any | Mgmt can't reach client devices |
| pass | MGMT net | any | any | Internet + infra access |

## WAN
| pass | 192.168.1.0/24 | 10.0.0.0/16 | any | senate LAN → homelab (static-route traffic) |

## NAT — Destination NAT
| Interface | Dest | Port | Target | Notes |
|---|---|---|---|---|
| WAN | WAN address | 25565 | 10.0.30.25:25565 | Minecraft → shipyard/Crafty (target confirmed at .25 — 2026-07-14 lease audit) |

## Design notes
- WAN "block private networks" is DISABLED — tarkin's WAN faces senate (192.168.1.0/24),
  not the raw internet.
- FAMILY/IOTGUEST exceptions are one-host-one-port pinholes above the broad 10.0.0.0/16
  block: DNS to order66 (10.0.30.53) and the Unbound fallback (10.0.30.1), and TCP 8096
  to cantina (Jellyfin — Phase 4, same pattern).

## Phase 3 DNS cutover — implemented 2026-07-13
FAMILY and IOTGUEST now pass port 53 to 10.0.30.53 (order66) and 10.0.30.1 (Unbound
fallback) above their blocks, as shown in the tables above; the old `→ 10.0.x.1 : 53`
exceptions were deleted after cutover verification. At the time the only inter-VLAN
hole — two hosts, one port (Phase 4 added the Jellyfin pinhole below). Details:
phases/phase-3-dns-and-tailscale.md.

## Phase 4 Jellyfin pinholes — implemented 2026-07
FAMILY and IOTGUEST pass TCP → 10.0.30.50:8096 (Jellyfin on cantina) above their
10.0.0.0/16 blocks, mirroring the DNS exception pattern. Verified in both directions
from both VLANs: allow — the Jellyfin app loads; deny — Portainer (10.0.30.25:9443)
remains unreachable. The inter-VLAN exceptions stay one-host-one-port pinholes: DNS +
Jellyfin, nothing broader. Details: phases/phase-4-cantina-and-gpu.md.