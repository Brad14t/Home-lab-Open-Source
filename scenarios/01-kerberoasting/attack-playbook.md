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

<img width="571" height="352" alt="image" src="https://github.com/user-attachments/assets/2a7910b6-e93b-478d-8eae-fca5974d7a5d" />

## Phase 2 — Request the service ticket

Now we actually ask for a ticket to the target service. Normally an attacker
would use a tool like Impacket's `GetUserSPNs` or Rubeus to do this and dump the
ticket in a crackable format in one step. In this lab, that classic approach is
dead on arrival — Windows Server 2025 disables the weak RC4 encryption those
tools rely on by default, even when the domain/account settings appear to allow
it (the RC4 key material is only regenerated on a password reset).

So instead, we use the **built-in Windows Kerberos cache tool** to request the
ticket directly — no extra tools, nothing that would need to touch disk or
trigger an AV signature. This is "living off the land": using what's already on
every domain-joined Windows machine.

```powershell
klist get MSSQLSvc/sql01.lab.local:1433
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `klist` | Built-in Windows tool for viewing and managing the current user's Kerberos ticket cache |
| `get` | Requests a new service ticket for the SPN given, rather than just listing cached tickets |
| `MSSQLSvc/sql01.lab.local:1433` | The exact SPN found during recon — this tells the Domain Controller which service account's ticket we want |

</details>

**Outcome:** The command succeeds and a new ticket for `svc-sql` is cached. This
is the critical moment — the Domain Controller just handed us a ticket encrypted
with the `svc-sql` account's password hash, to a completely unprivileged user,
with no alerts, no failed logins, and no special permissions required.

<img width="562" height="141" alt="image" src="https://github.com/user-attachments/assets/e9c3a2f0-c346-45cc-b785-d0e12aa8005e" />


**A note on what happens next in a real attack:** this ticket is encrypted with
AES, not RC4. AES-encrypted Kerberos tickets are computationally far more
expensive to crack offline than RC4 ones — RC4 can often be cracked in hours on
consumer hardware, while AES can take drastically longer depending on password
complexity. This lab environment's RC4 hardening means the classic "roast it in
an afternoon" version of this attack doesn't work here — but the exposure itself
(an unprivileged user obtaining any service account's ticket on demand) is
identical

## Phase 3 — Confirm possession of the ticket

A quick final check to prove the ticket is sitting in our cache, ready to be
used (or, in a less-hardened environment, extracted and cracked).

```powershell
klist
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `klist` | Lists every Kerberos ticket currently cached for this logon session — no arguments means "show, don't request" |

</details>

**Outcome:** The `svc-sql` ticket for `MSSQLSvc/sql01.lab.local:1433` appears in
the list, confirming we successfully hold a valid service ticket for an account
we have no legitimate reason to be touching.

<img width="601" height="393" alt="image" src="https://github.com/user-attachments/assets/a3300704-b779-4060-87ac-9cc8b7b80296" />

---

## Summary — what was actually achieved

As a standard, unprivileged domain user (`jcarter`), with no admin rights and no
attacker tooling beyond what Windows ships with:

1. Enumerated every Service Principal Name in the domain (`setspn`)
2. Identified `svc-sql` as a service account with an SPN
3. Requested and received a valid Kerberos service ticket for that account
   (`klist get`)
4. Confirmed the ticket is cached and usable

This is the complete Kerberoasting technique (MITRE ATT&CK T1558.003). In this
lab, the ticket's AES encryption makes it impractical to crack offline with
consumer hardware — but the underlying exposure (any domain user can request a
ticket for any service account, at will, with no alerting by default) is present
regardless of encryption type. The next document — 
[`blue-team-analysis.md`](./blue-team-analysis.md) — picks this up from the
defender's side, with none of the above assumed as known.
