# Runbook — DHCP / Lease Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No leases on any subnet | dnsmasq is holding port 67 before Kea | Services → dnsmasq → disable (keep it disabled — guardrail) |
| Wrong DNS/router in the lease | Kea auto-collect populated bad values | Disable auto-collect on the subnet; set DNS/router manually |
| Client keeps the old lease after a change | Kea serving a cached lease for that MAC | `release/renew` often isn't enough — physically disconnect the NIC or wait for lease expiry to force a fresh DISCOVER |

Verify active leases and their values at OPNsense → Services → Kea DHCP → Leases.