## Phase 1 — Recon: finding AS-REP-roastable accounts

With `jcarter`'s credentials already in hand from the earlier compromise, the
next step is finding out which domain accounts are vulnerable to AS-REP
roasting — specifically, any account with Kerberos pre-authentication
disabled. Rather than guessing usernames, authenticated LDAP enumeration can
pull this list directly from Active Directory.

```
GetNPUsers.py lab.local/jcarter:<JCARTER_PASSWORD> -dc-ip 10.0.30.10
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `GetNPUsers.py` | Impacket tool that queries AD for accounts with the `DONT_REQ_PREAUTH` flag set, and can optionally request AS-REP hashes for them |
| `lab.local/jcarter:<JCARTER_PASSWORD>` | Authenticates the LDAP query as `jcarter` — domain/username:password syntax |
| `-dc-ip 10.0.30.10` | Points the tool at the domain controller to query, since DNS resolution isn't assumed from the ATTACKER VLAN |

</details>

Placeholder `evidence/07-asrep-recon.png`.

This returns a list of accounts in the domain with pre-authentication
disabled — in this lab, `svc-backup` shows up on that list. This confirms the
account is actually vulnerable before moving on to request its AS-REP hash.
