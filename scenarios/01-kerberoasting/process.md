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

### Running the attack (from Kali)

I used **Impacket's GetUserSPNs**, a standard tool that automates the request.

My attacker box can't use the domain's DNS, so first I added a hosts entry so the
tool can reach the domain controller by name:

```bash
echo "10.0.30.10  dc01.lab.local lab.local" | sudo tee -a /etc/hosts
```

Then I requested the service ticket, authenticating as a normal, non-admin domain
user — that's all Kerberoasting needs:

```bash
impacket-GetUserSPNs lab.local/<user>:'<password>' -dc-ip 10.0.30.10 -request -outputfile kerberoast.hashes
```

What each part means:
- `lab.local/<user>:<password>` — logging in as an ordinary domain user
- `-dc-ip 10.0.30.10` — points straight at the domain controller (the KDC)
- `-request` — actually pulls the crackable ticket, not just a list of accounts
- `-outputfile` — saves the ticket hash to a file

> Never commit real credentials. The `<user>`/`<password>` placeholders stay as
> placeholders in this repo.

### What came back

<!-- FILL AFTER RUNNING: the svc-sql row with SPN MSSQLSvc/sql01.lab.local:1433,
     and a hash beginning with $krb5tgs$23$ -->

The `$23$` at the start of the hash means **RC4** encryption — the weak setting that
makes this crackable. An attacker would now feed this file to an offline cracker
like Hashcat.

### The blue-team takeaway (what matters for detection)

The cracking is invisible, but the **ticket request is not**. The moment the tool
asked the domain controller for that ticket, Windows logged it as **Event ID 4769**,
and recorded that the ticket used **RC4** encryption. Legitimate modern tickets use
AES, so an RC4 service-ticket request stands out. That's the signal the next section
detects.
