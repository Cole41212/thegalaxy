# Telemetry & Logging

The SIEM (inquisitor / Wazuh, Phase 6) is only as good as what feeds it. This document
defines the log pipeline now so sources can start shipping early and accumulate history
before the SIEM exists.

## Principle
Centralize logs from every meaningful source into one place. Start with a syslog target;
add Wazuh agents where an OS allows. The earlier sources ship, the more history the SIEM
has to work with on day one.

That last sentence is preserved as the original intent and as the lesson: the early sink was
never stood up, so inquisitor starts with none of it. See the Phasing note below.

## Sources (priority order)
Methods below are as settled by [ADR 0020](../decisions/0020-agent-coverage-and-log-sources.md):
**agent where the OS is ours, syslog where it is not**.

| Source | Logs | Method |
|---|---|---|
| tarkin (OPNsense) | firewall **block** logs (pass logs excluded, ADR 0018), system, VPN, DHCP | remote syslog (TCP, `allowed-ips`) |
| order66 (Pi-hole) | per-client DNS queries + blocks + reply codes | Wazuh agent |
| phantom (Tailscale gateway) | auth, sshd, tailscaled state | Wazuh agent |
| Proxmox host (executor) | auth, kernel, task, cluster, smartd, postfix delivery status, **plus host DNS activity via journald** (see the gap below) | Wazuh agent |
| shipyard | host auth + Docker/container logs — holds the lab's only inbound WAN port forward | Wazuh agent + Docker log driver |
| vault (Immich) | host auth + Docker/container logs, Immich auth + admin actions, DB-dump freshness | Wazuh agent + Docker log driver |
| cantina (Jellyfin) | host auth, Jellyfin auth + playback | Wazuh agent |
| inquisitor (Wazuh) | own auth, manager/indexer/dashboard health, SMART on `ssd-inquisitor` | Wazuh agent (local) |
| archives (TrueNAS) | auth, SMART/ZFS events, scrub results, share access | remote syslog (no agent — managed root filesystem) |
| death-star (switch) | port/link, auth | **none** — accepted unmonitored; syslog capability unconfirmed |
| falcon | Windows Security event log — logons, account changes, Defender | Wazuh agent (no Sysmon until Phase 7) |
| scout | endpoint security events | Wazuh agent — Phase 7, with the attack range |

**Gap — executor's DNS queries never reach Pi-hole.** executor's resolver is senate
(192.168.1.1), not order66, deliberately: postfix must resolve `smtp.gmail.com`, and pointing
the hypervisor at a guest would let order66's failure suppress the alert about order66's
failure (network/dns-design.md, ADR 0021). So the per-client DNS visibility below has a hole in
it exactly where the hypervisor is, and host DNS visibility comes from **journald ingestion**
instead — a different source answering the same question. Stated here so it is a known
limitation of the pipeline rather than a silent one.

## Highest-value signals
- **Firewall block logs** — denied inter-VLAN attempts = early lateral-movement signal.
- **DNS query logs (Pi-hole)** — per-client; catches malware C2 / DNS exfil. This is why
  clients point at Pi-hole directly (see `dns-design.md`).
- **DNS reply codes (Pi-hole)** — a SERVFAIL-ratio spike means the resolver path is broken
  even when users notice nothing: the Phase 3 zero-upstream incident ran invisibly because
  clients lived on the DHCP fallback. Phase 6 candidate rule from that incident.
- **Auth logs** — failed/odd logins across hosts.
- **ZFS/SMART events** — drive health before failure. Demonstrated on 2026-08-02, when the
  weekly scrub faulted a drive on a latent unreadable sector (Phase 5). The same incident
  exposed the gap that matters more than the signal: Proxmox alert mail was not being
  delivered at all, so the pool-degraded notification was never received and the fault was
  found by hand days later. **An alert channel nobody has tested is not a control** —
  verifying delivery end to end is a Phase 6 prerequisite, not a nice-to-have. **Acted on
  2026-08-16** — the relay was rebuilt and proven with a real delivery; design in
  [ADR 0021](../decisions/0021-alerting-and-notification-path.md).
- **SMART on `ssd-inquisitor` — the SIEM monitoring its own storage.** The Wazuh data disk sits
  on the lab's most-worn drive (Crucial BX200: 11,318 hours, 44.7 TiB written). If that drive
  degrades, the system that would report the degradation is the system living on it, so this
  rule is the one signal where the monitor and the monitored are the same component. It must
  read **raw** attribute values, not normalized ones: every SMART threshold on that drive is
  `000`, so its self-assessment is structurally incapable of reporting failure and any tool
  reading normalized values reports a perfect disk indefinitely. Trigger: attribute 169 raw
  below 80, or any nonzero value in 5 / 181 / 182 / 199 (docs/hardware-inventory.md).

## Architecture
    sources → syslog/agents → inquisitor (Wazuh manager + indexer, 10.0.30.60)
- Wazuh data/indices on SSD #2 (`ssd-inquisitor`) so log growth never starves the NVMe.
- Dashboards surfaced on panel-1/panel-2 (Phase 6).

## Phasing note (dependency) — resolved 2026-08-17
Wazuh is Phase 6, but Phases 3–5 built the sources worth logging. The plan was to decide on
a lightweight interim syslog sink (or inquisitor early in syslog-only mode) when Phase 3
landed order66, so firewall/DNS/auth history would accrue before detection rules were
written. That decision was never taken, and Phases 3, 4, and 5 have all shipped without a
central sink — so none of that history exists.

**Resolved: the sink was never built, and Phase 6 accepts starting with no back-history.** The
open question above is closed by accepting the second branch. There is no way to generate three
phases of telemetry retroactively, and deferring the SIEM for months to accumulate a baseline
would mean continuing to run with no detection at all — a worse trade than starting without
history.

The consequence is a change of method, not of scope. Rules are tuned against **deliberately
triggered events** rather than a historical baseline: generate the thing the rule should catch,
capture the **real log line** it produced, and test the decoder and rule against that captured
line with `wazuh-logtest`, which shows the decoder that claimed it, the fields extracted, and
the rule that matched. What is genuinely lost is the ability to say "this is unusual for this
host" on day one — frequency- and baseline-derived rules wait for the lab to build its own
history, which begins accruing the moment agents connect. Signature-shaped rules do not wait.

## Retention
Settled by [ADR 0018](../decisions/0018-wazuh-deployment-model-and-sizing.md): an ISM policy —
rollover, then **delete at 90 days** — on a 200 GiB dedicated data disk, with archives indices
**disabled** (they store every event rather than only alerts, and would fill the disk in weeks).
Retention is sized to keep usage clear of OpenSearch's 95% flood-stage watermark, past which
indices flip to read-only and the SIEM stops recording while still appearing to run.

The earlier plan to archive older indices to the `holocron` pool is **superseded** by
[ADR 0019](../decisions/0019-siem-data-placement-and-backup.md): index data is deliberately
expendable — alerts are derived evidence and the source logs remain on the endpoints — so no
snapshot repository is registered and nothing is written to the pool. Revisit in Phase 7 if
attack-exercise detection history has to outlive the retention window.