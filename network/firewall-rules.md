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
| pass | FAMILY net | 10.0.40.1 | 53 | DNS to gateway — must sit above block |
| block | FAMILY net | 10.0.0.0/16 | any | No homelab access |
| pass | FAMILY net | any | any | Internet only |

## IOTGUEST (VLAN 50)
| Action | Source | Destination | Port | Notes |
|---|---|---|---|---|
| pass | IOTGUEST net | 10.0.50.1 | 53 | DNS to gateway — must sit above block |
| block | IOTGUEST net | 10.0.0.0/16 | any | No homelab access |
| pass | IOTGUEST net | any | any | Internet only |

## SECLAB (VLAN 60)
| block | SECLAB net | 10.0.0.0/16 | any | Fully isolated from homelab |
| pass | SECLAB net | any | any | Internet (to be tightened in Phase 7 — rogue gets none) |

## MGMT (VLAN 10)
| block | MGMT net | TRUSTED net | any | Mgmt can't reach client devices |
| pass | MGMT net | any | any | Internet + infra access |

## WAN
| pass | 192.168.1.0/24 | 10.0.0.0/16 | any | senate LAN → homelab (static-route traffic) |

## NAT — Destination NAT
| Interface | Dest | Port | Target | Notes |
|---|---|---|---|---|
| WAN | WAN address | 25565 | 10.0.30.25:25565 | Minecraft → shipyard/Crafty |

## Design notes
- WAN "block private networks" is DISABLED — tarkin's WAN faces senate (192.168.1.0/24),
  not the raw internet.
- FAMILY/IOTGUEST DNS exception exists because the broad 10.0.0.0/16 block would otherwise
  drop DNS to the gateway.

## Pending change — Phase 3 DNS cutover (see network/dns-design.md)
When clients move to Pi-hole (10.0.30.53), the FAMILY and IOTGUEST port-53 exceptions
change from `10.0.x.1` to `10.0.30.53` (optionally keep `10.0.x.1` as a fallback). This is
the only new inter-VLAN hole — one host, port 53.