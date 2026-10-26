# Blue Team Analysis — Threat Hunt & Detection Gap Investigation

> **Perspective:** This document is written from the **defender's** point of view.
> I had **no prior knowledge** of the attack described in
> [`attack-playbook.md`](./attack-playbook.md) going into this. Unlike a typical
> alert-driven investigation, this one starts from a **proactive hunt** — because,
> as the investigation itself will show, the automated detection didn't fire the
> way it should have.

## Step 1 — Why hunt instead of wait for an alert?

Kerberoasting is a technique that, by design, leaves almost no trace in the
places defenders usually look first. There's no failed login, no account
lockout, no obviously malicious process — just a Kerberos ticket request that
looks, on the surface, like completely normal domain activity. A detection rule
can be written for it, but relying on that rule alone is risky: if it's wrong,
misconfigured, or has a blind spot, the attack can walk right past it with
nothing to show for it afterward.

For that reason, I periodically **hunt** for Kerberoasting indicators directly,
rather than waiting for an alert — treating any configured detection rule as a
safety net, not the only line of defense. I started this hunt with one simple
question: *which service accounts in this domain have had Kerberos service
tickets requested for them recently, and by whom?*

## Step 2 — Pulling the data

In the Wazuh dashboard, under **Discover**, I searched raw Windows Security
events (not just alerts) for Kerberos service ticket requests:

```
data.win.system.eventID:4769
```

<details>
<summary>What this search does</summary>

| Part | What it does |
|---|---|
| `data.win.system.eventID:4769` | Filters to only Windows Event ID 4769 — "A Kerberos service ticket was requested" — across all ingested logs, alerted or not |

</details>

This surfaced a high volume of routine results: machine accounts (`DC01$`)
requesting their own tickets, and `krbtgt` renewal traffic. That's expected,
constant background noise on any domain. To cut through it, I added a second
filter targeting only **user-initiated** requests to accounts that *look like*
service accounts:

```
data.win.system.eventID:4769 and data.win.eventdata.serviceName:svc-*
```

This narrowed the view to requests for any service account following the
lab's `svc-` naming convention — and one result stood out:

| Field | Value |
|---|---|
| `targetUserName` | jcarter@LAB.LOCAL |
| `serviceName` | svc-sql |
| `ticketEncryptionType` | 0x12 (AES256) |
| `ipAddress` | ::ffff:10.0.30.102 |

<img width="760" height="448" alt="image" src="https://github.com/user-attachments/assets/ad573bfd-9c98-455f-a31e-c47920fc24c4" />

<img width="588" height="440" alt="image" src="https://github.com/user-attachments/assets/53fb6ff2-e487-4a3f-b9ae-b4758cd202d6" />

`0x12` is AES256-CTS-HMAC-SHA1-96 — the modern, strong Kerberos ticket
encryption type. That's expected here.

**Why this stood out:** `jcarter` is a standard end-user account. `svc-sql`
is a service account that should only ever be used by whatever system actually
runs the SQL service — never requested directly by a person's own logon. A
user's workstation asking for a ticket to a service account it has no business
touching is exactly the kind of anomaly Kerberoasting produces.

## Step 3 — Checking for a corresponding alert

A suspicious raw event is a lead, not a conclusion. Before treating this as a
confirmed incident, I checked whether this activity had already tripped a
detection — because if it did, there should be a corresponding alert sitting
in the queue.

I switched from raw event data to **alerts only**:

```
data.win.system.eventID:4769 and rule.id:(100130 or 100131)
```

<details>
<summary>What this search does</summary>

| Part | What it does |
|---|---|
| `data.win.system.eventID:4769` | Still scoped to Kerberos service ticket requests |
| `rule.id:(100130 or 100131)` | Filters to only events that matched one of the two custom Kerberoasting detection rules written for this lab — rule `100130` (RC4 ticket encryption) and rule `100131` (monitored service account name) |

</details>

**Result: zero matches for the `jcarter → svc-sql` event.**

<img width="761" height="340" alt="image" src="https://github.com/user-attachments/assets/7491019d-3fef-4785-b4db-9a65ed1ac1a9" />

## Step 3b — Confirming this alert isn't the one I was looking for

The one record that came back for `rule.id:100131` fired, but pulling apart
its fields showed me three things that ruled it out as evidence of the real
attack:

| # | Field | What it showed | Why it rules this alert out |
|---|---|---|---|
| 1 | `targetUserName` | `adm-brad@LAB.LOCAL` | This is a different account entirely from the one I found during the hunt (`jcarter@LAB.LOCAL`). `adm-brad` is an admin account I used earlier to validate the rule logic, not the account that touched `svc-sql` in the real attack. |
| 2 | `ipAddress` | `::1` | This is the IPv6 loopback address — the request originated from the Wazuh/DC itself, not from a workstation on the network. The real attack traffic came from `::ffff:10.0.30.102` (WS01, on the CORPORATE VLAN). A loopback-sourced request can't be the remote attack I'm hunting. |
| 3 | `systemTime` | `2026-10-25T20:57:57Z` | This timestamp lines up with when I was testing the detection rule, not with the actual `jcarter` → `svc-sql` request I found in the Discover hunt. |

**Conclusion:** this alert is a leftover from rule testing — proof the rule
*can* fire, but not evidence it fired for the real attack. The `jcarter`
event I found in Step 2 still has no corresponding alert anywhere in Wazuh.

This is the gap. I had a detection rule written for exactly this pattern — a
monitored service account (`svc-sql`) being requested outside of normal
machine-account activity — and it simply didn't fire for the real attack
traffic. I confirmed this wasn't a fluke by checking whether the rule was even
active and loaded:

```
data.rule.id:100131
```

This search *did* return a result — just not the one that mattered. It showed
a single alert, generated earlier during rule testing, for a different,
benign account (`adm-brad`) making a request from `localhost` rather than from
an actual workstation on the network. So the rule was live, loaded, and
capable of firing — it just wasn't firing for events coming from a real remote
host.

**At this point it became a detection engineering problem, not just a threat
hunting one.** A rule that works in testing but not in production is arguably
worse than no rule at all — it creates false confidence that Kerberoasting
activity will be caught automatically, when in practice it won't be. The next
step was figuring out exactly why.

## Step 4 — Detection engineering investigation

A rule that fires in testing but not for the real attack traffic means the
rule itself isn't necessarily broken — something about the *production*
conditions is different from the *test* conditions. I needed to find that
difference.

### 4a — Re-confirming the rule logic is sound

First, I re-validated the rule syntax directly on the manager to rule out a
config error:

```
sudo /var/ossec/bin/wazuh-analysisd -t
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `sudo` | Required — only root can read Wazuh's config/rule files |
| `/var/ossec/bin/wazuh-analysisd` | The Wazuh analysis engine binary — the component that evaluates rules against incoming events |
| `-t` | "Test configuration" mode — validates all rules and decoders without starting the service, and reports syntax errors or load failures |

</details>

This came back clean — no errors, correct rule count loaded. The rule file
itself was not the problem.

### 4b — Isolating the discrepancy to the source IP format

I compared the one working test alert against the real attack event,
field-by-field, using the raw indexed archive data rather than relying on the
dashboard:

```
sudo grep '"eventID":"4769"' /var/ossec/logs/archives/archives.json | jq -c 'select(.data.win.eventdata.targetUserName=="jcarter@LAB.LOCAL")'
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `sudo grep '"eventID":"4769"' /var/ossec/logs/archives/archives.json` | Searches Wazuh's full event archive (every event ingested, not just alerts) for Kerberos ticket requests |
| `jq -c 'select(.data.win.eventdata.targetUserName=="jcarter@LAB.LOCAL")'` | Parses each matching line as JSON and filters down to only the event where the requesting account is `jcarter` — compact one-line-per-record output |

</details>

This confirmed the real event *was* ingested and correctly decoded by Wazuh —
every field was present and well-formed. The only meaningful difference
between this event and the one that triggered rule `100131` during testing was
the `ipAddress` field:

| | Test event (fired) | Real attack event (did not fire) |
|---|---|---|
| `ipAddress` | `::1` | `::ffff:10.0.30.102` |

`::1` is plain IPv6 loopback. `::ffff:10.0.30.102` is an **IPv4-mapped IPv6
address** — the standard notation a dual-stack system uses to represent an
IPv4 address inside an IPv6 field. Same rule, same decoder, same event
structure — just a different IP address *format*.

### 4c — Ruling out the obvious alternatives

Before concluding it was the IP format, I ruled out the more common causes of
"rule works in testing but not in production":

- **EPS throttling** — raised the agent's `max_eps` from 50 to 500 on DC01 and
  re-tested. No change.
- **Event backlog / delayed forwarding** — checked `ossec.log` timestamps
  against the agent's connection state; the event arrived and was processed
  in real time, not queued.
- **Manager crash or restart mid-event** — checked `ossec.log` for
  SIGNAL/restart entries around the event's timestamp; none occurred. The
  same `wazuh-analysisd` process that fired the test alert was still running
  when the real event came through.
- **Rule not actually loaded** — confirmed via the "Total rules enabled"
  count in `ossec.log`, which incremented correctly after every edit.

None of these explained it. The IP address format was the one consistent
variable.

### 4d — Likely root cause

Further research into how Wazuh internally parses IP address fields pointed to
a known class of issue: Wazuh's internal field-extraction logic for IP
addresses can rely on a character-class pattern that doesn't account for the
embedded `.` characters inside an IPv4-mapped IPv6 address like
`::ffff:10.0.30.102`. If that internal extraction step silently stops or
mismatches partway through the value, it can affect downstream rule
processing for that event — without logging any visible error.

This matches the external reporting on Wazuh's handling of IPv6 traffic
generally: see [Wazuh IPv6 Security Monitoring — oneuptime.com](https://oneuptime.com/blog/post/2026-03-20-wazuh-ipv6-security-monitoring/view).

**Bottom line:** the detection rule is logically correct and does fire for a
`svc-sql` Kerberos ticket request — confirmed in Step 3b. But in this
environment, Windows Server 2025 reports the client address of *any* remote
(non-loopback) caller in IPv4-mapped IPv6 notation, and that notation appears
to interfere with Wazuh's event matching for this rule in a way that doesn't
affect loopback-sourced test traffic. This is a genuine detection gap, not a
one-off fluke.
