# 0020 — Agent coverage and log sources: agent where the OS is ours, syslog where it is not
**Status:** Accepted (Phase 6, 2026-08-17)

**Context:** docs/telemetry-logging.md lists what is worth logging. This ADR settles *how*
each source reaches inquisitor, which is a different question with different consequences: an
agent is software installed as root on the monitored host, and syslog is an unauthenticated
datagram from anywhere. Choosing wrongly either creates an unsupportable dependency on an
appliance's root filesystem, or throws away FIM and inventory on a host that could have had
them.

**Decision:** Wazuh agents on **inquisitor, shipyard, vault, cantina, order66, phantom,
executor, and falcon**. Remote syslog for **tarkin** and **archives**. No agent on TrueNAS.
falcon runs the Windows agent **without Sysmon**. **death-star** is recorded as an accepted
unmonitored source.

### Sources and what each contributes

| Source | Method | What it contributes |
|---|---|---|
| inquisitor (Wazuh manager) | agent (local) | Its own auth and sudo, manager/indexer/dashboard service health, and SMART on `ssd-inquisitor` — the SIEM monitoring its own storage (ADR 0018, docs/telemetry-logging.md) |
| executor (Proxmox host) | agent | Hypervisor auth, sudo, kernel, PVE task and job logs, smartd, postfix delivery status, **and host DNS activity via journald** — executor resolves through senate, so its queries never appear in Pi-hole (network/dns-design.md) |
| tarkin (OPNsense) | remote syslog | Firewall **block** logs (pass logs excluded, ADR 0018), system, VPN, Kea/DHCP — the lateral-movement signal the segmentation exists to produce |
| order66 (Pi-hole) | agent | Per-client DNS queries and blocks, plus **reply codes** — the SERVFAIL-ratio rule seeded by the Phase 3 zero-upstream incident |
| archives (TrueNAS) | remote syslog | Auth, ZFS pool and scrub events, SMART, share access — the source of the 2026-08-02 pool-degraded event that was never delivered |
| phantom (Tailscale gateway) | agent | sshd, auth, and tailscaled state on the lab's only tailnet ingress (ADR 0011) |
| shipyard (Docker host) | agent | Host auth plus Docker/container logs on the host holding the lab's **only inbound WAN port forward** — the most externally exposed system in the lab |
| vault (Immich) | agent | Host auth, Docker logs, Immich authentication and admin actions, and local freshness checks on the nightly database dump (ADR 0016) |
| cantina (Jellyfin) | agent | Host auth, Jellyfin authentication and playback — the one service FAMILY and IOTGUEST can reach through a pinhole (Phase 4) |
| falcon (Windows) | agent | Windows Security event log: logons, failed logons, account and group changes, Defender detections. **No Sysmon** |
| death-star (switch) | none | Accepted unmonitored — see below |

**Alternatives:**

*An agent on archives* (rejected) — TrueNAS SCALE is Debian underneath, which makes an agent
technically installable and operationally wrong. It is an appliance with a managed root
filesystem: an update replaces it, and third-party packages installed outside the middleware
do not survive. The failure would be silent, and you would discover it mid-incident when the
host you most wanted telemetry from turns out to have stopped reporting weeks ago. Phase 5
already produced the matching lesson on the same host — the `/etc/environment` PATH edit
carries the same "may not survive an update" caveat.

*An agent on tarkin* (rejected) — FreeBSD-based OPNsense has no supported Wazuh agent. Syslog
is the vendor-supported path and is what the appliance is built to emit.

*Sysmon on falcon now* (deferred to Phase 7) — Sysmon's value is process-creation, network-
connection, and image-load telemetry, and that telemetry is only interesting when something is
generating it deliberately. Phase 7 stands up scout as the attack box; installing Sysmon now
means tuning a high-volume source against a workstation doing nothing but ordinary work.

*Syslog from death-star* (unresolved, accepted) — the ZX-SWTGW215AS manual's chapter structure
shows no obvious log-host or remote-syslog section, so it is not yet established that the
switch can export anywhere at all. Recorded as an accepted unmonitored source rather than as a
task with an assumed outcome; the actual question is whether the capability exists, and that
gets confirmed in the switch UI before anything is planned around it.

**Consequences:**

### The split as a principle

**Agent where you control the OS; syslog where you do not.** The line is ownership of the root
filesystem, not the operating system's family or the host's importance. Where a general-purpose
OS is ours to manage, an agent buys far more than log shipping. Where the host is a vendor
appliance whose root filesystem is managed by its own update process, an agent is a dependency
that will be removed without warning, so the appliance's own supported export path is the
correct one even though it yields strictly less.

### What an agent actually is

The Wazuh agent is a set of cooperating components, and the teaching value is in knowing which
one produces which signal:

- **logcollector** — tails files, reads the Windows event log, and accepts journald.
- **syscheck (FIM)** — file integrity monitoring: hashes watched paths and reports create,
  modify, and delete events.
- **rootcheck** — policy and anomaly checks for rootkit-like artefacts.
- **syscollector** — hardware, OS, package, port, and process inventory; this is what
  vulnerability detection consumes when it is eventually enabled (ADR 0018).
- **SCA** — Security Configuration Assessment: benchmark checks against the running system.

That list explains why **the agent necessarily runs as root**. FIM must read files no
unprivileged user may read, inventory must enumerate every process and listening socket, and
log collection must read `/var/log/auth.log` and its equivalents. There is no least-privilege
version of this component that still does its job.

### Enrolment and transport

Enrolment is a key exchange, not a login. The agent contacts **authd on 1515/TCP** once,
authenticates with the enrolment password, and receives a pre-shared key that is written into
`client.keys` on both ends. Every subsequent connection uses that key on **1514/TCP**, and
1515 is only reachable during enrolment. `client.keys` is therefore a secret and is backed up
accordingly (ADR 0019).

**The agent always dials out.** The manager never initiates a connection to an agent. No
monitored host needs an inbound port opened for monitoring, and in particular **falcon on
VLAN 20 needs no firewall pinhole** — its agent originates the session toward 10.0.30.60, and
TRUSTED → SERVERS is already permitted by the documented design. This is worth stating because
the intuitive mental model of "the monitoring server polls the endpoints" would have produced a
rule change on every VLAN.

### Agents collect; they do not analyse

Decoding and rule evaluation happen **centrally**, on inquisitor. The agent ships events; it
holds no detection logic. Two consequences follow. First, detection logic lives in exactly one
place, so a rule is written once and applies everywhere rather than being deployed to twelve
hosts and drifting. Second, **a compromised endpoint cannot disable its own rules** — the best
it can do is stop reporting, and stopping reporting is itself an event: the manager's
agent-status rules (501/502/503) fire on connection state changes, so silence from an agent
becomes an alert rather than an absence.

### Syslog's weaknesses, stated rather than assumed

Syslog is unauthenticated and trivially spoofable — anything that can reach the listener can
inject a line claiming to come from tarkin. Three mitigations follow directly: the manager's
syslog listener is restricted with **`allowed-ips`** so only tarkin's and archives' addresses
are accepted; **TCP is preferred over UDP**, since UDP is connectionless, silently lossy, and
even easier to forge; and it must be remembered that syslog yields **log lines only** — no FIM,
no SCA, no inventory, no vulnerability data. The two syslog hosts are therefore permanently
less observable than the agent hosts, by an amount that is a property of the choice and not a
gap to be closed later.

### Blast radius: inquisitor is a crown-jewel host

This is the uncomfortable consequence and it is recorded rather than glossed. The manager can
push configuration to every agent, and every agent runs as root. Compromise of inquisitor
therefore yields **root on every monitored host in the lab**, plus the ability to delete the
evidence of how it happened. In blast-radius terms inquisitor now sits alongside executor and
tarkin, and it is *newly* in that category — the lab did not previously have a component with
lab-wide root reach. Mitigations, all structural rather than procedural:

- **`remote_commands` stays disabled** (the default). This is what prevents a manager-pushed
  `agent.conf` from executing arbitrary commands on agents, and it is precisely why the Immich
  database-dump freshness check is configured **locally on vault** rather than pushed centrally.
  The inconvenience of local configuration is the visible price of a real control.
- **Active response is disabled globally.** Automatic blocking in a lab whose router is a VM on
  the same host as the SIEM is a self-inflicted lockout waiting to happen: a false positive
  that null-routes or firewalls the wrong address takes out management access to the machine
  that would undo it. Revisit in Phase 7, where the attack range gives a safe place to test it.
- **authd uses an enrolment password**, not open registration. Open registration lets anything
  that can reach 1515/TCP enrol itself and receive a key.
- **The manager is reachable from VLAN 30 and the tailnet only** — not from FAMILY, IOTGUEST,
  or SECLAB.

### Identity note — three names on one host

executor carries three identities simultaneously, all deliberate:

- its **OS hostname is `pve`** (the Proxmox install default; renaming a node means relocating
  `/etc/pve/nodes/pve/` and fixing storage and job references for no functional gain),
- its **lab name and Wazuh agent name are `executor`** (agent names are arbitrary and set at
  enrolment, so the SIEM can use the name the documentation uses), and
- its **agent connects from 10.0.30.2**, the VLAN 30 storage leg, because that leg is directly
  connected to the same bridge as inquisitor and does not transit tarkin (ADR 0009).

Expect all three to appear in different places — Proxmox UI, Wazuh agent list, and alert source
addresses — and do not treat the mismatch as a misconfiguration.
