---
name: repo-audit
description: Audit the thegalaxy homelab repo for consistency, accuracy, and gaps. Use when asked to audit, verify, review, or sanity-check the repo — typically after committing a phase.
---
# Repo Audit
1. Read source-of-truth docs first: docs/hardware-inventory.md, network/ip-scheme.md,
   network/firewall-rules.md, network/dns-design.md, docs/, decisions/.
2. Check cross-file consistency: hardware specs, IP assignments, VM inventory, and
   status markers must agree across README, phase docs, diagrams, and docs/.
3. Known gotchas: all statics below .100 (pools .100-.200); no duplicated paragraphs or
   contradictory status lines; diagrams match hardware (512GB NVMe, 2 NICs, 12 drives,
   dual controllers); no secrets committed; config-backups/ contents gitignored.
4. For version-specific GUI procedures (Proxmox 9.x, TrueNAS SCALE CE, OPNsense 26.x),
   verify against current docs rather than memory.
5. Output: prioritized findings (file path + section + fix), then write the fixes as
   file edits for the owner to review in GitHub Desktop. Never commit.
