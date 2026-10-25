# Attack Playbook — Kerberoasting (T1558.003)

> **Perspective:** This document is written from the **attacker's** point of view.
> It describes how the attack is carried out. It contains **no detection or defensive
> content** — that lives in [`blue-team-analysis.md`](./blue-team-analysis.md), written
> from the defender's point of view with no prior knowledge of what's here.

## What Kerberoasting is (in plain terms)

In an Active Directory network, every "service" (like a database or web app) that
runs under a dedicated account has a label called a **Service Principal Name (SPN)**
— basically a tag that says "this account runs this service here."

When any user wants to talk to that service, Active Directory hands them an encrypted
**service ticket**. Here's the catch: that ticket is encrypted with the service
account's password. So **any logged-in user can ask for a ticket for any service** —
and then try to crack that ticket offline to recover the service account's password.

That's Kerberoasting: ask for the ticket, take it away, and crack the password at your
leisure — no alarms from failed logins, because you never guess passwords against the
network. You only need to be a **normal domain user** to do it.

---

## Phase 1 — Recon: find accounts worth roasting

The first step is finding which accounts have SPNs. We only care about **user**
accounts with SPNs (service accounts), not computer accounts — those are the
crackable targets.

We use `setspn`, a tool that ships with Windows. No special tools, no admin rights.

```powershell
setspn -T lab.local -Q */*
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `setspn` | Built-in Windows tool for viewing and editing Service Principal Names in Active Directory |
| `-T lab.local` | **T**arget: search within the `lab.local` domain |
| `-Q */*` | **Q**uery for SPNs matching `*/*` — the first `*` is any service type, the second is any host, so this returns **every** SPN in the domain |

</details>

**What to look for in the output:** most entries will be computer accounts (their names
end in `$`, e.g. `DC01$`). Ignore those. The valuable targets are SPNs registered to a
**regular user account** — in this lab, that's:

```
MSSQLSvc/sql01.lab.local:1433
    Registered to: CN=svc-sql,CN=Users,DC=lab,DC=local
```

`svc-sql` is a user account pretending to be a SQL Server service. That's our target.

![SPN recon output](evidence/01-spn-recon.png)
