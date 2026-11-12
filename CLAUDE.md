# CLAUDE.md — thegalaxy
Rules for any AI session working in this repo — a phase chat editing the working tree
directly, or Claude Code. Full write-up in docs/conventions.md.

## What this is
A Star Wars-themed cybersecurity homelab, documented as a blue-team portfolio.
The owner is a Marist cybersecurity senior who wants to understand WHY, not just steps.

## Source of truth (read before answering)
- docs/hardware-inventory.md — hardware, controllers, drive mapping, IOMMU groups
- network/ip-scheme.md — IPs; network/firewall-rules.md — rules; network/dns-design.md — DNS
- docs/ and decisions/ — rationale and ADRs; phases/ — build logs
Authoritative. Trust them over conversation; if something conflicts, flag it.

## How to work
- WHY before HOW: reasoning and tradeoffs, then steps.
- Technical, but define terms in passing. No redundancy or over-caution.
- Do not re-explain steps marked done.
- GUI-first for lab config (Proxmox/TrueNAS/OPNsense); CLI only when no GUI path — say why.
- Read screenshots/specs precisely — never assume counts; ask if ambiguous.
- Troubleshooting: reason from evidence, state confidence, most decisive test first.

## Editing this repo
- Edits are written straight into the working tree, uncommitted. Before the first edit of a
  session, confirm the working tree matches the pushed commit; never overwrite a file that
  changed since it was read.
- Every change: file path + section heading + the change.
- Phase docs: Decisions & Rationale, Checklist (emoji), Verification evidence, dated Log
  (Issue → Root cause → Fix → Learning), Deferred / follow-ups.
- Cross-cutting decisions → numbered immutable ADRs in decisions/. Never edit the decision
  text: supersede with a new ADR, or append a dated `## Amendment`. List decisions/ before
  citing or assigning an ADR number — never cite one that has no file.
- No inline TODO markers in published docs; an open question is a Deferred item.
- Star Wars naming; devices as hostname (name — what it is).
- NEVER commit secrets. config-backups/ contents are gitignored; device XML/keys stay out.
- NEVER write personal location detail: no place names, state or region, no Wi-Fi network
  names (an SSID geolocates), no public WAN address, no tailnet 100.x addresses, no ISP plan
  or account detail. Sites are "the primary site" / "the offsite location". Redact silently —
  never add a sentence describing what was removed. Rules: docs/conventions.md.
- NEVER commit, push, or run git on the owner's behalf. Write files; the owner reviews the
  diff and commits. Nothing enters history unread.

## Operational guardrails (never violate)
- dnsmasq stays disabled (port 67 vs Kea).
- Kea auto-collect OFF; set DNS/router manually per subnet.
- Save switch config to flash BEFORE applying.
- tarkin (VM 100) and archives (VM 102) back up ONLY to ssd-vmstore (local). Never to
  the NAS — the NAS path depends on tarkin for routing and archives for storage itself.
- nas-vmbackups NFS mount stays soft,timeo=30,retrans=3 — never change to hard.
- Hung NFS recovery: umount -f -l /mnt/pve/nas-vmbackups
- Check executor's memory budget BEFORE creating any VM (qm list, free -h). Passthrough
  guests are pinned, so pressure lands on the router and DNS first.
- After any host power event: archives up and holocron imported with 12 disks, before
  anything else.
- Pi-hole's upstream is exactly 10.0.30.1 (Unbound on tarkin); Pi-hole's DHCP stays off.
- Never add wazuh-states-* indices to an age-based retention policy — they hold current
  state, not history.
