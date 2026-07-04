# CLAUDE.md — thegalaxy
Rules for working in this repo. Full write-up in docs/conventions.md.

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
- Every change: file path + section heading + the change.
- Phase docs: Decisions & Rationale, Checklist (emoji), dated Log
  (Issue → Root cause → Fix → Learning).
- Cross-cutting decisions → numbered immutable ADRs in decisions/ (supersede, never edit).
- Star Wars naming; devices as hostname (name — what it is).
- NEVER commit secrets. config-backups/ contents are gitignored; device XML/keys stay out.
- NEVER commit or run git on the owner's behalf. Write files; the owner reviews in
  GitHub Desktop and commits.

## Operational guardrails (never violate)
- dnsmasq stays disabled (port 67 vs Kea).
- Kea auto-collect OFF; set DNS/router manually per subnet.
- Save switch config to flash BEFORE applying.
- Infrastructure VMs (tarkin 100, archives 102) back up ONLY to local storage
  (ssd-vmstore), never to the NAS. The NAS path must never depend on the VM being
  backed up.
