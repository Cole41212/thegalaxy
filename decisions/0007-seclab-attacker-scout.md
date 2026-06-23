# 0007 — SECLAB attacker is the scout laptop, not a Kali VM
**Status:** Accepted (planned, Phase 7); supersedes the maul VM
**Context:** Wanted an attack box for the security lab; also want offensive tooling
portable on a real machine.
**Decision:** scout (Linux Mint laptop) joins VLAN 60 via a tagged/access switch port as the
attacker. rogue (vulnerable VM) stays fully isolated with NO internet, reachable only from
scout. The maul Kali VM is dropped from the plan.
**Alternatives:** Keep maul as a VM (another VM on the NVMe; tools not portable).
**Consequences:** One fewer VM; portable tooling. SECLAB firewall policy finalized in
Phase 7 (rogue → WAN blocked; scout ↔ rogue allowed within VLAN 60).