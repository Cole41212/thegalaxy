# Project Conventions

How The Galaxy is designed, built, and documented. These conventions keep the project
consistent and make the AI-assisted workflow reproducible.

## Working principles
- **WHY before HOW.** Every decision is justified before the steps are given.
- **GUI-first.** Lab configuration (Proxmox, TrueNAS, OPNsense) is done through the GUI
  to build real tooling fluency; the command line is used only where there's no GUI path,
  and the reason is stated.
- **Evidence over assumption.** Hardware facts are read from the actual screen/output,
  never assumed. Ambiguity is flagged, not guessed.
- **Decisive troubleshooting.** Reason from evidence, state confidence, run the most
  decisive test first, and update fast as facts arrive.

## Source of truth
The repo is authoritative. Current state lives in:
- `docs/hardware-inventory.md` — hardware, controllers, drive mapping, IOMMU groups
- `network/ip-scheme.md` — IP plan
- `network/firewall-rules.md` — firewall policy
- `network/dns-design.md` — DNS topology
- `docs/` and `decisions/` — design rationale and ADRs
- `phases/` — per-phase build logs

## Documentation standards
- Every repo change names the **file path + section heading + the change**.
- **Phase docs** follow three parts: Decisions & Rationale, Checklist (emoji status),
  and a dated Log where problems use Issue → Root cause → Fix → Learning.
- **Cross-cutting decisions** are recorded as ADRs in `decisions/` — numbered and
  immutable (superseded by a new ADR, never edited).
- **Naming** is Star Wars throughout; devices are referred to as hostname (name — what
  it is), e.g. "tarkin (OPNsense firewall)".

## Security & secrets
- Device configuration exports (OPNsense XML, switch config) and any keys **never** go in
  the public repo. `config-backups/` is gitignored; real exports live offsite.

## Workflow
- Documentation is written in bulk at phase completion.
- Commits: `Phase N: <what>` for phase work, `Repo: <what>` for maintenance.
- Git via GitHub Desktop; Git Bash when the command line is needed.
- A new chat is used per phase; after committing and resyncing, a repo audit verifies
  consistency before the next phase begins.

## AI-assisted workflow
Claude (Anthropic) is used as a pair programmer and documentation partner. The owner makes
the architecture decisions and performs all hands-on configuration and verification on real
hardware; AI accelerates research, sanity-checks designs, and helps produce documentation.
Build logs preserve the reasoning, including independently diagnosed faults.