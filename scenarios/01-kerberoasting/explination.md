## Scenario 01 — Kerberoasting

### What this attack is (in plain terms)

Active Directory uses a system called **Kerberos** to let users prove who they are
without sending their password across the network every time. Instead of passwords,
it hands out encrypted **tickets**.

Some accounts in a domain are **service accounts** — accounts that run background
services like a database. Each is tagged with an **SPN** (Service Principal Name),
a label that says "this account runs this service."

Here's the weakness: **any** logged-in domain user — even a low-privilege one — is
allowed to ask the domain controller for a service ticket for any SPN. That ticket
is encrypted with the service account's password. If the account uses older **RC4**
encryption, an attacker can take the ticket back to their own machine and try
millions of password guesses against it **offline**, where no defender can see it.

This technique is **Kerberoasting** (MITRE ATT&CK **T1558.003**). Attackers like it
because:
- It only needs one ordinary domain account to start.
- The password cracking happens offline, so it's silent.
- Service accounts often have weak, never-changing passwords and high privileges.

> **Why document the attack in a blue-team project?** I can't build a good detection
> for something I don't understand. The goal isn't the attack — it's seeing exactly
> what trace it leaves so I can catch it.

### The target

| | |
|---|---|
| Account | `svc-sql` |
| SPN | `MSSQLSvc/sql01.lab.local:1433` |
| Weakness | RC4 encryption, weak password that never expires |

This account was **deliberately misconfigured** in the lab to be roastable, so there's
something realistic to detect.


### The blue-team takeaway (what matters for detection)

The cracking is invisible, but the **ticket request is not**. The moment the tool
asked the domain controller for that ticket, Windows logged it as **Event ID 4769**,
and recorded that the ticket used **RC4** encryption. Legitimate modern tickets use
AES, so an RC4 service-ticket request stands out. That's the signal the next section
detects.
