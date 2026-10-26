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
