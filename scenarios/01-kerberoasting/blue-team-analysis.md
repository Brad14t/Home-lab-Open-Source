# Blue Team Analysis — Threat Hunt & Detection Gap Investigation

> **Perspective:** This document is written from the **defender's** point of view.
> The analyst working this has **no prior knowledge** of the attack described in
> [`attack-playbook.md`](./attack-playbook.md). Unlike a typical alert-driven
> investigation, this one starts from a **proactive hunt** — because, as the
> investigation itself will show, the automated detection didn't fire the way it
> should have.

## Step 1 — Why hunt instead of wait for an alert?

Kerberoasting is a technique that, by design, leaves almost no trace in the
places defenders usually look first. There's no failed login, no account
lockout, no obviously malicious process — just a Kerberos ticket request that
looks, on the surface, like completely normal domain activity. A detection rule
can be written for it, but relying on that rule alone is risky: if it's wrong,
misconfigured, or has a blind spot, the attack can walk right past it with
nothing to show for it afterward.

For that reason, mature SOCs periodically **hunt** for Kerberoasting indicators
directly, rather than waiting for an alert — treating any configured detection
rule as a safety net, not the only line of defense. This hunt starts with one
simple question: *which service accounts in this domain have had Kerberos
service tickets requested for them recently, and by whom?*

## Step 2 — Pulling the data

In the Wazuh dashboard, under **Discover**, the analyst searches raw Windows
Security events (not just alerts) for Kerberos service ticket requests:

```
data.win.system.eventID:4769
```

<details>
<summary>What this search does</summary>

| Part | What it does |
|---|---|
| `data.win.system.eventID:4769` | Filters to only Windows Event ID 4769 — "A Kerberos service ticket was requested" — across all ingested logs, alerted or not |

</details>

This surfaces a high volume of routine results: machine accounts (`DC01$`)
requesting their own tickets, and `krbtgt` renewal traffic. That's expected,
constant background noise on any domain. To cut through it, the analyst adds a
second filter targeting only **user-initiated** requests to accounts that *look
like* service accounts:

```
data.win.system.eventID:4769 and data.win.eventdata.serviceName:svc-*
```

This narrows the view to requests for any service account following the lab's
`svc-` naming convention — and one result stands out:

| Field | Value |
|---|---|
| `targetUserName` | jcarter@LAB.LOCAL |
| `serviceName` | svc-sql |
| `ticketEncryptionType` | 0x12 (AES256) |
| `ipAddress` | ::ffff:10.0.30.102 |

<img width="767" height="447" alt="image" src="https://github.com/user-attachments/assets/33c32dbf-630c-4bd1-a74c-a8093fd77b43" />


**Why this stands out:** `jcarter` is a standard end-user account. `svc-sql`
is a service account that should only ever be used by whatever system actually
runs the SQL service — never requested directly by a person's own logon. A
user's workstation asking for a ticket to a service account it has no business
touching is exactly the kind of anomaly Kerberoasting produces.
