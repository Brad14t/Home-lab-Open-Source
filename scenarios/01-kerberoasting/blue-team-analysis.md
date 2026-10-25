# Blue Team Analysis — Alert Investigation

> **Perspective:** This document is written from the **defender's** point of view.
> The analyst working this case has **no prior knowledge** of the attack described
> in [`attack-playbook.md`](./attack-playbook.md) — everything below is reasoned
> from what the monitoring tools actually show, the way a real SOC investigation
> would unfold.

## Step 1 — The alert

While reviewing the Wazuh dashboard, an alert appears:

> **Possible Kerberoasting (monitored service account): svc-sql ticket requested
> by jcarter@LAB.LOCAL from ::ffff:10.0.30.102**
> Rule 100131 · Level 10

Level 10 is high enough to warrant immediate attention — this isn't routine noise.
Before assuming anything, the first job is to understand exactly what this alert
is telling us and whether it's actually suspicious.

![Wazuh alert detail](evidence/04-wazuh-alert-fired.png)

---

## Step 2 — Understand the raw event

Clicking into the alert's full event data in Wazuh shows the underlying Windows
Event 4769 (Kerberos Service Ticket Request) that triggered it:

| Field | Value | What it means |
|---|---|---|
| `eventID` | 4769 | A Kerberos **service ticket** was requested — not a login, a ticket for a *service* |
| `targetUserName` | jcarter@LAB.LOCAL | The account that **requested** the ticket |
| `serviceName` | svc-sql | The account the ticket was **issued for** — i.e., whose password-derived key encrypts it |
| `ticketEncryptionType` | 0x12 | AES256 — a modern, hard-to-crack encryption type (0x17 would mean the weaker RC4) |
| `ipAddress` | ::ffff:10.0.30.102 | Where the request came from |

**First read:** `jcarter` is a standard user. `svc-sql` sounds like a service
account — those normally only get ticket requests from the *servers* that run the
service, not from a random user's workstation. That mismatch is the core of why
this is suspicious: **why would a regular user's machine be requesting a ticket
for a SQL service account at all?**

This single event, on its own, proves someone *asked* for the ticket. It doesn't
yet prove intent — plenty of
