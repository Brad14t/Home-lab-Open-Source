# Scenario 01 — Kerberoasting

> Requesting a service account's Kerberos ticket as an ordinary domain user, to crack
> its password offline. This lab shows the classic version being blocked by modern AD
> hardening, the attack succeeding via a living-off-the-land method instead, and a
> Wazuh detection that catches it regardless of encryption.

| | |
|---|---|
| **MITRE ATT&CK** | T1558.003 — Steal or Forge Kerberos Tickets: Kerberoasting |
| **Tactic** | Credential Access |
| **Target** | `svc-sql` — SPN `MSSQLSvc/sql01.lab.local:1433` |
| **Attacker start** | `atk-kali-01` · 10.0.50.10 (classic attempt) |
| **Attacker (successful)** | `WS01` · 10.0.30.102, as compromised user `jcarter` |
| **Detection source** | Wazuh custom rules `100130`/`100131` on Windows Event 4769 |
| **Status** | ✅ Detected (AES variant, encryption-agnostic rule) |
| **Time to detect** | `<attack time>` → `<alert time>` (near real-time) |

---

## What this attack is (in plain terms)

Active Directory uses **Kerberos** to prove who users are without sending passwords
across the network. Instead of passwords, it issues encrypted **tickets**.

Some accounts run background services (a database, a backup job). Each is tagged with
an **SPN** (Service Principal Name) — a label saying "this account runs this service."
The weakness: **any** logged-in domain user can ask the domain controller for a
service ticket for any SPN, and that ticket is encrypted with the service account's
password. The attacker takes it to their own machine and guesses the password
**offline**, where no defender can see the guessing.

This is **Kerberoasting**. Attackers favor it because it needs only one ordinary
account, the cracking is silent, and service accounts often have weak, never-changing
passwords with high privileges.

> **Why a blue-team project documents the attack:** I can't detect what I don't
> understand. The goal is to see the exact trace the attack leaves so I can catch it.

---

## The target

| | |
|---|---|
| Account | `svc-sql` |
| SPN | `MSSQLSvc/sql01.lab.local:1433` |
| Weakness | Service account with an SPN and a crackable password |

`svc-sql` was deliberately created with an SPN so it would be roastable. There's no
real database behind `sql01.lab.local` — it's a decoy that exists to be attacked and
detected.

---

## Prerequisites & scope

- **Temporary firewall rule:** `atk-kali-01` (10.0.50.10) → `DC01` (10.0.30.10) on
  TCP 88 (Kerberos) and 389 (LDAP), for the initial Kali attempt. Disabled before
  and after the run.
- **Logging:** Kerberos service-ticket auditing (Event 4769) is enabled domain-wide
  via GPO, and `DC01` runs a Wazuh agent forwarding the Security log.
- **VM set booted:** 100 (pfSense), 101 (Wazuh), 110 (DC01), 120 (WS01), 140 (Kali).

> **Teardown:** the temporary Kali→DC rule is disabled immediately after the run.

---

## Attack, part 1 — the classic RC4 roast (and why it failed)

The standard Kerberoast uses **Impacket's GetUserSPNs** from Kali. Because the
attacker box can't use the domain's DNS, I first mapped the domain controller by name.

From Kali:

```bash
echo "10.0.30.10  dc01.lab.local lab.local" | sudo tee -a /etc/hosts
```

<details>
<summary>Command explained</summary>

| Component | What it does |
| :--- | :--- |
| `echo "10.0.30.10 dc01.lab.local lab.local"` | One host-file line: `[IP] [hostname] [alias]`, mapping the DC's IP to its names. |
| `\| sudo tee -a /etc/hosts` | Appends (`-a`) that line to the system hosts file with admin rights. |

</details>

Then I requested the service ticket, authenticated as an ordinary domain user
(`jcarter`):

```bash
impacket-GetUserSPNs lab.local/jcarter:'<password>' -dc-ip 10.0.30.10 -request
```

<details>
<summary>Command explained</summary>

| Component | What it does |
| :--- | :--- |
| `impacket-GetUserSPNs` | Finds accounts with SPNs and can request their tickets. |
| `lab.local/jcarter:'<password>'` | Logs in as an ordinary domain user — no admin rights needed. |
| `-dc-ip 10.0.30.10` | Points at the domain controller directly (no DNS dependency). |
| `-request` | Actually requests the crackable ticket, not just a list. |

</details>

**Result:** the tool found `svc-sql` and its SPN, but the ticket request was rejected:

```
[-] Principal: lab.local\svc-sql - Kerberos SessionError: KDC_ERR_ETYPE_NOSUPP (KDC has no support for encryption type)
```

<img width="479" height="73" alt="image" src="https://github.com/user-attachments/assets/ddff7ceb-91cc-4dda-a8fd-d8860fb7d758" />

### Why it failed (the finding)

`KDC_ERR_ETYPE_NOSUPP` means the domain controller refused to issue an **RC4**-encrypted
ticket. Impacket requests RC4 by default (RC4 is the easiest to crack), but **Windows
Server 2025 disables RC4 by default.** I tried to force the classic attack through:

- Set the account to allow RC4: `Set-ADUser svc-sql -KerberosEncryptionType RC4,AES128,AES256`
- Allowed RC4 at the domain-controller policy ("Network security: Configure encryption
  types allowed for Kerberos")
- Reset the account password so its keys would regenerate

**It still failed.** RC4 is deprecated deeply enough in modern AD that re-enabling it
at the account level, the policy level, and rotating the password was not enough to
get the KDC to issue an RC4 ticket. 

**Takeaway:** a modern domain resists the textbook RC4 Kerberoast by default. An
attacker has to fall back to AES — and, importantly, a detection built only around
"RC4 is bad" would be blind to what comes next.

---

## Attack, part 2 — the working AES roast (living off the land)

Instead of attacker tooling, a service ticket can be requested with **built-in Windows
functionality** from any domain workstation. This is how a real attacker on a
compromised machine operates — no tools to install, nothing for antivirus to catch —
and it automatically uses AES, which the domain *does* support.

From `WS01`, logged in as the compromised user `jcarter`:

```
klist get MSSQLSvc/sql01.lab.local:1433
```

<details>
<summary>Command explained</summary>

| Component | What it does |
| :--- | :--- |
| `klist` | Built-in Windows tool for viewing and managing Kerberos tickets. |
| `get MSSQLSvc/sql01.lab.local:1433` | Requests and caches a service ticket for that SPN. Requesting it **is** the Kerberoast action — the DC logs Event 4769. |

</details>

**Result:** the DC issued the ticket, encrypted with AES-256
(`KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96`). The attack succeeded against
the hardened domain — just with a cipher that's slower (but not impossible) to crack.

<img width="370" height="91" alt="image" src="https://github.com/user-attachments/assets/1f0486fe-2f61-411c-821d-39496ad9e4e2" />

---

## Detection

Event 4769 fires on the domain controller for *every* service-ticket request —
thousands a day, almost all legitimate. A useful detection has to separate the attack
from that noise. Two signals do it:

1. **RC4 encryption (`0x17`)** — the classic tell. Kept as a high-severity rule, though
   in this hardened domain it won't fire (the lesson from part 1).
2. **Requests for sensitive service accounts** — an ordinary user should never need a
   ticket for `svc-sql` or `svc-backup`. Any 4769 naming them, *regardless of
   encryption*, is worth an alert. This is what catches the AES attack.

### Confirming Wazuh saw the event

With full logging enabled temporarily, the raw 4769 event showed the attack's
fingerprints:

```json
"serviceName": "svc-sql", "ticketEncryptionType": "0x12",
"targetUserName": "jcarter@LAB.LOCAL", "ipAddress": "::ffff:10.0.30.102"
```

(`0x12` = AES256; `10.0.30.102` = WS01.)

<img width="478" height="195" alt="image" src="https://github.com/user-attachments/assets/4097ee01-5f83-40af-9c70-763788e2a753" />

### The detection rules

Added to Wazuh's `local_rules.xml`:

```xml
<group name="windows,kerberoasting,">

  <rule id="100130" level="12">
    <if_sid>60106</if_sid>
    <field name="win.system.eventID">^4769$</field>
    <field name="win.eventdata.ticketEncryptionType">^0x17$</field>
    <description>Possible Kerberoasting (RC4 ticket): $(win.eventdata.serviceName) requested by $(win.eventdata.targetUserName) from $(win.eventdata.ipAddress)</description>
    <mitre><id>T1558.003</id></mitre>
  </rule>

  <rule id="100131" level="10">
    <if_sid>60106</if_sid>
    <field name="win.system.eventID">^4769$</field>
    <field name="win.eventdata.serviceName" type="pcre2">^(svc-sql|svc-backup)$</field>
    <description>Possible Kerberoasting (monitored service account): $(win.eventdata.serviceName) ticket requested by $(win.eventdata.targetUserName) from $(win.eventdata.ipAddress)</description>
    <mitre><id>T1558.003</id></mitre>
  </rule>

  <rule id="100139" level="5">
    <if_sid>60106</if_sid>
    <field name="win.system.eventID">^4769$</field>
    <description>TEST - any Kerberos 4769 ticket request</description>
  </rule>

</group>
```

| Rule | Fires when | Level | Catches |
| :--- | :--- | :--- | :--- |
| `100130` | 4769 + RC4 (`0x17`) | 12 | Classic RC4 roast (won't fire in this AES-hardened domain) |
| `100131` | 4769 naming `svc-sql`/`svc-backup` | 10 | The AES attack run here — encryption-agnostic |

After deploying the rules and re-running the attack, rule `100131` fired at level 10,
naming `svc-sql`, `jcarter`, and the source workstation.

<img width="171" height="45" alt="image" src="https://github.com/user-attachments/assets/61e59e68-6c7b-4f45-a710-d6351aaf8ff3" />

<img width="767" height="244" alt="image" src="https://github.com/user-attachments/assets/ae890777-6980-4887-a9f7-0fd860197560" />

---

## Analysis

- **The modern default already helped the defender.** The textbook RC4 attack was
  blocked outright. The attack only worked by moving to AES — which cracks far slower,
  buying defenders time.
- **Encryption-agnostic detection matters.** A rule that only looks for RC4 would have
  missed this entirely. Watching for *who* is asking for *which* service account is the
  more durable signal.
- **The attack was toolless.** The successful request used only built-in Windows
  commands, so endpoint antivirus had nothing to flag. The detection had to come from
  the domain controller's own logs — which is exactly why SIEM visibility into 4769
  matters.
- **This was a single request.** A real roast often pulls *many* service accounts at
  once. A strong follow-up detection is a frequency rule: one user requesting tickets
  for many distinct service accounts in a short window (a future addition).

---

## Remediation & hardening

- **Use group Managed Service Accounts (gMSAs)** for service accounts. They use
  long (120-character), automatically-rotated passwords that are effectively
  uncrackable.
- **Keep RC4 disabled** (it already is here) and enforce AES — it slows cracking
  dramatically.
- **Give service accounts long, random passwords** (25+ characters) if gMSA isn't an
  option, so even an AES ticket is impractical to crack.
- **Least privilege:** service accounts should not be in privileged groups — a cracked
  low-privilege account is far less damaging.
- **Detect, don't just prevent:** alert on 4769 for sensitive service accounts, on RC4
  ticket requests anywhere, and on one user requesting many SPNs quickly.

---

## References

- [MITRE ATT&CK T1558.003 — Kerberoasting](https://attack.mitre.org/techniques/T1558/003/)
- Microsoft — [4769(S, F): A Kerberos service ticket was requested](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4769)
