# Runbook — Firewall (tarkin) Recovery

When a rule change locks you out or breaks routing.

## Access the firewall out-of-band
Proxmox → VM 100 (tarkin) → Console. This reaches OPNsense over the VM's virtual console,
independent of the network — so a bad firewall rule can't lock you out of it.

## Emergency: a rule is blocking your own access
From the OPNsense console → shell:
    pfctl -d      # disables the packet filter temporarily (re-enables on reboot)
Regain GUI access at https://10.0.20.1, fix the offending rule, then:
    pfctl -e      # re-enable
Never leave PF disabled — it's an open firewall.

## Restore a known-good config
GUI: System → Configuration → Backups → Restore (upload the last-good XML).
Console: option 13 (restore a backup) if the GUI is unreachable.

## Last resort: factory reset
Console → "Reset to factory defaults", then reassign interfaces (WAN = vtnet0,
LAN trunk = em0) and restore config from backup.

## Key facts
- GUI: https://10.0.20.1 (Trusted) — also reachable from Management.
- WAN faces senate (192.168.1.0/24); "block private networks" stays DISABLED.