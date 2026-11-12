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
- **Phase docs** follow five parts: Decisions & Rationale, Checklist (emoji status),
  Verification evidence, a dated Log where problems use Issue → Root cause → Fix → Learning,
  and Deferred / follow-ups. Wrong predictions are recorded in the log as wrong, not reshaped.
- **Cross-cutting decisions** are recorded as ADRs in `decisions/` — numbered and
  immutable. The decision text is never edited: a decision that changes is superseded by a
  new ADR, and what was learned implementing one is appended as a dated `## Amendment`.
- **Open questions are written as tracked follow-ups**, in a Deferred section, not left as
  inline `TODO` markers in a published document.
- **Naming** is Star Wars throughout; devices are referred to as hostname (name — what
  it is), e.g. "tarkin (OPNsense firewall)".

## Security & secrets
- Device configuration exports (OPNsense XML, switch config) and any keys **never** go in
  the public repo. `config-backups/` is gitignored; real exports live offsite.
- **No personal location detail.** No place names, no state or region, no street or building
  detail, and no Wi-Fi network names past or present — an SSID can be looked up to a location.
  Describe a site by its role ("the primary site", "the offsite location").
- **No account or addressing detail that identifies the connection:** no public WAN address,
  no tailnet (100.64.0.0/10) addresses, no ISP plan tier or account detail.
- **Redact silently.** When something is removed, do not add a sentence saying what was
  removed or why — the explanation advertises exactly what was taken out. Where a command in
  a log used a sensitive string as its argument, write `PATTERN` in its place.
- **Text search cannot audit binaries.** Images, PDFs and certificates need looking at
  directly; a certificate is handed to every client that connects.

## Operational guardrails
The standing rules that came out of incidents. The short form lives in `CLAUDE.md`; the
reasoning lives in the phase log or ADR named beside each.
- **Check the host memory budget before creating a VM**, not after it misbehaves. RAM on
  executor is a shared, finite resource. Two things make overcommitting it unusually costly:
  archives and cantina use PCI passthrough, so their guest RAM is pinned and all host memory
  pressure lands on the remaining guests; and the kernel evicts the least-recently-used pages,
  so idle infrastructure — the router, DNS, the tailnet gateway — is swapped first and is the
  worst thing to have stalled. `qm list` for allocation, `free -h` for real headroom.
  (Phase 6, 2026-09-24; ADR 0018 amendment.)
- **After any host power event, verify archives is up and `holocron` is imported with all 12
  disks before anything else.** A hard NFS mount on a missing pool produces hangs elsewhere
  that look nothing like a storage fault. (Phase 4; runbooks/site-relocation.md.)
- **Infrastructure VMs never back up to storage that depends on them.** tarkin and archives
  back up to local `ssd-vmstore` only. (ADR 0009.)
- **Let a vendor installer install stock, confirm it works, then adapt it.** Configuration
  staged for software that does not exist yet cannot be validated against anything. (Phase 6.)
- **Open every multi-command step with `hostname`.** (Phase 6 — commands were run against the
  wrong host twice.)

## Workflow
- One chat per phase. Lab work, troubleshooting and documentation all happen in that chat.
- Documentation is written in bulk at phase completion; a mid-phase checkpoint is fine when
  the repo has drifted far enough from the lab to mislead.
- Commits: `Phase N: <what>` for phase work, `Repo: <what>` for maintenance.
- Git from the command line (Git Bash on falcon). The owner reviews every diff and makes
  every commit and push.
- Two clones exist (falcon and scout). `git fetch` and `git status` before a session on
  either; never pull blind.

## AI-assisted workflow
Claude (Anthropic) is used as a pair programmer and documentation partner. The owner makes
the architecture decisions and performs all hands-on configuration and verification on real
hardware; AI accelerates research, sanity-checks designs, and drafts the documentation.
Build logs preserve the reasoning, including independently diagnosed faults.

**How a documentation change reaches this repo:**

1. **The chat is linked to the owner's computer, with access to this repo's folder only.**
   Claude reads the pushed repository and the working tree, and can write files into the
   working tree. It has no shell there and cannot run git.
2. **Before editing, Claude compares the working tree with the pushed commit**, so local work
   is never overwritten unseen, and each write is refused if the file changed since it was
   read.
3. **Claude writes the edits directly into the working tree, uncommitted**, and reports every
   change as file path + section + what changed.
4. **The owner reads the diff** (`git diff`, and the new files directly), runs the
   sanitization check on the staged tree, then commits and pushes.

The line that does not move: **nothing enters this repo's history without the owner reading
it.** Claude never commits, never pushes, and never runs git on the owner's behalf. An edit
that is wrong is one `git restore` away from gone.

This replaced an earlier arrangement (through November 2026) in which the chat produced a
written prompt and a separate Claude Code session carried it out. The hand-off was the weak
point: instructions were executed at one remove from the evidence, and the gaps showed up as
drift — an ADR cited that had no file, a runbook that was specified and never created, a
sanitization step that did not take. Editing in the same chat that holds the evidence removes
the hand-off; the owner's review of the diff is unchanged.

`CLAUDE.md` holds the short-form rules any AI session must follow when editing this repo, and
`.claude/skills/repo-audit/` holds the consistency audit run after a phase is committed.
