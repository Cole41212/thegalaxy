# 0021 — Alerting and the notification path
**Status:** Accepted (Phase 6, 2026-08-17)

**Context:** On 2026-08-02 a drive in holocron faulted during the weekly scrub. Proxmox
generated the alert correctly. Nobody received it, and the fault was found by hand days later.
The cause was not a missing rule or a missing sensor — every layer of detection worked. The
delivery path was broken and had been for months, and nothing about the lab's behaviour
distinguished "no alerts because nothing is wrong" from "no alerts because the mail queue is
full." That is the failure this ADR exists to make structurally impossible, and it is why the
notification work was ordered *ahead* of the SIEM build in Phase 6 rather than after it.

This ADR is the phase's central decision. Detection rules written against an unproven channel
are unverifiable by construction: a rule that fires into a black hole and a rule that never
fires look identical from the outside.

---

## Principles

### 1 — An untested channel is not a control

A notification path counts as working when a **deliberately triggered real event** has been
observed arriving at the destination device, end to end. It does not count when a vendor's
"Send test message" button reports success. Those buttons exercise a different code path than
the real alert takes — they frequently bypass the queue, the template, the sender rewriting,
and the routing matcher, which is to say they bypass most of the places the path actually
breaks. The 2026-08-16 postfix incident is the concrete proof: every *sending program* on
executor was succeeding. cron, smartd, and PVE hand a message to postfix and exit zero. The
break was one layer below all of them, and no test that stopped at "the command succeeded"
could ever have found it.

Consequence: **every path is re-tested quarterly**, by triggering something real, not by
pressing test. A control that was verified once and then left alone for a year is a control
whose current state is unknown.

### 2 — No alert path may share a failure domain with what it monitors

An alert about a failure must not travel through the failing thing. This is the same rule that
ADR 0009 applied to backups — never back up the router through the router — restated for
notification.

Applied here, it rejects a **self-hosted notification server on executor outright**. A push
gateway, a Gotify, an ntfy instance: any of them, hosted on the hypervisor, would be unable to
deliver the single most important alert in the lab, which is "executor is down." It equally
rejects routing lab alerts through tarkin: executor's mail leaves via vmbr0 → senate → the ISP
and never transits the firewall VM, so a tarkin outage cannot suppress the alert reporting it.

It is the same principle that keeps **executor's resolver on senate (192.168.1.1) rather than
order66** — postfix needs DNS to resolve `smtp.gmail.com`, and pointing the hypervisor at a
guest would mean order66 going down could suppress the alert saying order66 is down (see
network/dns-design.md).

### 3 — Absence must be converted into a positive signal

Silence is ambiguous. It means "nothing is wrong" and it means "the alerting is dead," and on
2026-08-02 those two states were indistinguishable — which is exactly why the outage lasted
months rather than a day. Any monitoring design that only emits on bad news inherits this
ambiguity permanently.

The fix is a heartbeat: a component that is *expected* to check in, so that its failure to
check in is itself the alert. This inverts the dependency — the alert about the alerting
system does not travel through the alerting system.

### 4 — Tier by severity, or you train yourself to ignore it

The realistic failure mode of a home lab SIEM is not missing one alert. It is receiving so many
that reading them stops being a habit, and then missing everything. The economics are
asymmetric and worth stating numerically: **one false page costs a night's sleep; ten false
pages cost you the system**, because after ten you stop looking, and the system is then worth
less than no system at all — it produces the *belief* that you are covered.

So severity is assigned deliberately, P3 is the default, and moving something up a tier
requires an argument written down.

---

## Three transports, independent by construction

Three channels, chosen so that no two fail together.

### Email — the workhorse (P2 and P1)

Four senders, deliberately not funnelled through one relay:

- **executor** runs postfix as an **authenticated satellite relay** to `smtp.gmail.com:587`.
  Rebuilt 2026-08-16; the full diagnosis and configuration are in the Phase 6 log. The short
  version: `relayhost` was empty, so postfix attempted direct-to-MX delivery on port 25, which
  the ISP blocks, and 20 messages spanning several days sat deferred.
- **PVE** additionally gets a **native SMTP notification target** in Datacenter →
  Notifications, which talks to Gmail directly and **bypasses postfix entirely**. Two
  independent paths off the same host: if one is misconfigured, the other still delivers.
- **archives (TrueNAS)** uses its **own native email** configuration rather than relaying
  through executor — an appliance with a supported mail path should use it, and routing NAS
  alerts through the hypervisor would couple the storage alert path to the hypervisor.
- **Wazuh** uses a **local postfix satellite on inquisitor**. This is not stylistic: Wazuh's
  built-in mailer speaks plain, unauthenticated SMTP and cannot do STARTTLS or SASL, so it
  cannot talk to Gmail at all. Handing off to a local postfix that *can* is the supported
  pattern, and it also means inquisitor's mail configuration is inspectable with the same tools
  as executor's.

### Push — Pushover, for P1 only

P1 alerts go to the phone by push. **Pushover** rather than self-hosted **ntfy**, and the
reasoning is a hardware fact rather than a preference: a self-hosted push server **cannot
deliver to iOS**. Apple Push Notification service will only accept notifications signed with a
certificate tied to a registered application, which means only Apple — or the app vendor —
can push to an iPhone. A self-hosted ntfy instance targeting iOS ends up relaying through
ntfy.sh's own infrastructure anyway, which negates the one advantage self-hosting was supposed
to buy. Given that, running the server is pure cost: another container, another dependency, and
— by principle 2 — one that would sit inside the failure domain it reports on.

The deciding functional difference is **emergency priority**: Pushover repeats an emergency
notification until it is explicitly acknowledged. That matters for exactly the alert class where
the recipient is asleep, which is exactly the P1 class. A notification that arrives silently at
3 a.m. and is buried by morning has not been delivered in any sense that counts.

### Deadman — external, and permanently so

**Hourly pings** from **inquisitor** and **executor** to an external dead-man's-switch service
(the same one already carrying the offsite replication timers from Phase 5), with a **90-minute
grace period** — one full missed interval plus slack, so a single delayed ping does not page.

**Two pings rather than one**, because two sources are diagnostic where one is only binary:

| Observed | Meaning |
|---|---|
| Both pinging | Host and SIEM both alive |
| Neither pinging | executor is down, or the site has lost power or internet |
| executor pinging, inquisitor not | **SIEM-specific failure** — the host is fine, the monitoring is not |
| inquisitor pinging, executor not | Implausible; investigate as a routing or ping-configuration fault |

The third row is the one that justifies the second ping. It is the state that was invisible for
months in 2026, and with one ping it stays invisible.

**The deadman stays external in Phase 8 too.** When the lab acquires a domain and could
plausibly self-host a status page, it must not host this. A deadman hosted inside the lab it
monitors is not a deadman — it is a component that goes down with everything else, at which
point silence means what it always meant.

---

## Severity tiers

| Tier | Transport | Response | Assigned to |
|---|---|---|---|
| **P1 — page** | Pushover **emergency** (repeats until acknowledged) **+ email** | Now, including overnight | ZFS pool DEGRADED or FAULTED on holocron or carbonite · deadman grace expired for executor or inquisitor · tarkin down / lab-wide routing or DHCP loss · Wazuh manager or indexer service down · successful authentication that should not exist (root SSH on executor, a new device joining the tailnet, a new agent enrolling with authd) |
| **P2 — notify** | Email | Same day | vzdump job **failure** · Wazuh agent disconnected beyond 15 minutes · SMART replacement trigger crossed on `ssd-inquisitor` or `ssd-vmstore` · no Immich database dump in 48h (ADR 0016) · offsite replication miss · repeated failed authentication (brute-force shape) on any host · SERVFAIL-ratio spike on order66 (Phase 3 seed) · block-log spike originating from a **lab** host — lateral-movement candidate · scrub or resilver completing with errors · postfix deferred-queue depth non-zero for more than an hour |
| **P3 — record** | Indexed only; read on the dashboard | On review | **Everything else** — routine authentication, successful backups, Jellyfin playback, DNS queries and blocks, package updates, SCA results, ordinary blocks of low-trust VLANs at the firewall |

**P3 is the default, and it must hold the overwhelming majority of events.** A rule is written
at P3 unless there is a stated reason otherwise; promotion to P2 or P1 requires an argument
recorded alongside the rule, answering what the recipient would *do* differently on receiving
it. "It seems important" is not an argument. If the answer is "look at it tomorrow," it is P2;
if the answer is "nothing, but it is good to know," it is P3 and belongs on a dashboard.

Note the deliberate asymmetry in the P2 row for the firewall: block-log spikes from **low-trust
VLANs** are P3 — that is the segmentation working as designed, every day. A block-log spike
originating from a **lab** host is P2, because a server on VLAN 30 probing addresses it has no
business probing is the lateral-movement signal the whole design exists to surface.

---

## Consequences

**Immediate, and it will bite within a week if unhandled.** PVE's `notification-mode` is
`notification-system`, which routes every notification through the matchers in Datacenter →
Notifications rather than mailing `root@pam` directly. With the relay now working, the **two
nightly vzdump jobs will deliver success reports every single night** — the NAS job and the
local infrastructure job (ADR 0009). Two guaranteed emails a day is precisely the shape of
noise that principle 4 warns about, and it arrives from the very system whose alerts were just
restored. A matcher must therefore route backup **failures** to P2 and **suppress successes**,
before the first nightly run lands. Suppressing them is safe here specifically *because* the
deadman covers the "did the job run at all" question from outside; without that, silence on a
backup job would be ambiguous again.

**Testing is now part of the build, not part of the write-up.** Each of the three transports is
proven by a real triggered event and the evidence recorded in the Phase 6 verification section:
for mail, the receiving server's acceptance code (`250 2.0.0 OK … gsmtp`) rather than the
sending command's exit status; for push, the notification arriving and the emergency retry
behaving as documented; for the deadman, a deliberately skipped ping producing an alert after
the grace period.

**Cost accepted.** Pushover is a one-time paid licence per platform, and the dead-man's-switch
service is a third party that learns the lab's uptime pattern and the names of its checks. Both
are accepted: the licence is trivial against the value of an alert that repeats until
acknowledged, and the deadman's whole purpose requires it to be outside the lab, which
necessarily means someone outside the lab holds that metadata. No log content, no addresses,
and no configuration leave the lab through either channel.

**Revisit triggers.** (a) A P1 that pages and turns out not to have needed paging — retune
immediately, per principle 4, rather than tolerating it. (b) Phase 8 introducing a public
service, which adds an alert class this tiering does not yet cover. (c) Any quarterly re-test
that fails, which reopens the design rather than merely fixing the instance.
