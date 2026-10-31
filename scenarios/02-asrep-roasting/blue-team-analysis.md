# Blue Team Analysis — AS-REP Roasting Detection

> **Perspective:** This document is written from the **defender's** point of
> view. I had no prior knowledge of the attack described in
> [`attack-playbook.md`](./attack-playbook.md) going into this. Unlike the
> Kerberoasting investigation, this one starts with a gap that gets fully
> closed — a rule is written, and then proven to work against real attack
> traffic.

## Step 1 — Checking existing detection coverage

Before assuming any detection gap, I checked whether Wazuh already had
visibility into AS-REP requests at all. AS-REP roasting shows up as **Event ID
4768** ("A Kerberos authentication ticket (TGT) was requested") rather than
4769, which the existing Kerberoasting rules were built around.

In Wazuh Discover, I searched:

```
data.win.system.eventID:4768
```

This returned **nothing at all** — not even routine noise like machine-account
TGT renewals. That absence was itself informative: for comparison, 4769
events always showed up in Discover because a built-in Wazuh rule (part of
the default Windows logon-success chain) already matches that event ID, even
when nothing suspicious is happening. No equivalent built-in rule exists for
4768. Unlike the Kerberoasting case, where a rule existed and simply failed
to fire reliably, **there was no detection path for AS-REP roasting here at
all — built-in or custom.**

I confirmed this wasn't a forwarding/indexing issue by checking the raw event
archive directly on the Wazuh manager rather than relying on the dashboard:

```
sudo grep '"eventID":"4768"' /var/ossec/logs/archives/archives.json
```

## Step 2 — Building the detection rule

With confirmation that no rule existed for Event ID 4768, I wrote one — but
applied two specific lessons from the earlier Kerberoasting investigation
rather than repeating the same mistakes.

**Lesson 1 — anchoring.** During the Kerberoasting investigation, I found that
Wazuh's rule engine appears to evaluate only the *first matching child* under
a given parent rule, silently skipping any sibling rule added alongside an
already-matching one. The fix there was anchoring the custom rule directly to
the specific endpoint rule that was already matching, rather than making it a
sibling competing for the same parent. I applied the same principle here:
traced the built-in rule chain this event already passes through
(`60001` → `60103`, since the event's `severityValue` is `AUDIT_SUCCESS`) and
confirmed no existing child of `60103` claims Event ID 4768. Anchoring the new
rule there as a child, rather than as a sibling of something else under
`60001`, avoids the same silent-skip problem before it can happen.

**Lesson 2 — the IP field.** The Kerberoasting rule's remote-traffic gap was
ultimately traced to matching a `<field>` condition directly against the
`ipAddress` value, which is reported in IPv4-mapped IPv6 notation
(`::ffff:x.x.x.x`) for any non-loopback source. This time, the rule only
*references* `ipAddress` inside the description text — it's never used as a
match condition — so that bug has no surface to affect this rule.

The actual vulnerability indicator for AS-REP roasting is the
`Pre-Authentication Type` field reporting `0`, meaning the KDC issued a TGT
without requiring any proof of password knowledge first — exactly what
happens when an account has Kerberos preauthentication disabled.

```xml
<group name="windows,asreproasting,">

  <rule id="100132" level="12">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4768$</field>
    <field name="win.eventdata.preAuthType">^0$</field>
    <description>Possible AS-REP Roasting: TGT requested without Kerberos pre-authentication for $(win.eventdata.targetUserName) from $(win.eventdata.ipAddress)</description>
    <mitre>
      <id>T1558.004</id>
    </mitre>
  </rule>

</group>
```

<details>
<summary>Rule explained</summary>

| Part | What it does |
|---|---|
| `<if_sid>60103</if_sid>` | Anchors this rule as a child of the built-in rule that already matches any successful Windows security event, avoiding the sibling-skip issue found during the Kerberoasting investigation |
| `<field name="win.system.eventID">^4768$</field>` | Matches only Kerberos TGT request events |
| `<field name="win.eventdata.preAuthType">^0$</field>` | Matches only requests where no Kerberos pre-authentication was performed — the actual AS-REP roasting indicator |
| `<description>...$(win.eventdata.targetUserName)...$(win.eventdata.ipAddress)</description>` | Dynamically fills in which account was targeted and where the request came from — `ipAddress` is used here only for display, never as a match condition |
| `<mitre><id>T1558.004</id></mitre>` | Tags the alert with the correct MITRE ATT&CK sub-technique for AS-REP Roasting |

</details>

After saving the rule, I validated the syntax and reloaded the manager:

```
sudo /var/ossec/bin/wazuh-analysisd -t
```
```
sudo systemctl restart wazuh-manager
```

Both completed cleanly, confirming the rule loaded without error.

Once a fresh AS-REP request was generated, this confirmed the event *was*
being ingested and fully decoded by Wazuh (all fields present and correct) —
it just had zero rule coverage, so it never reached the alerts index that
Discover searches by default.

**Conclusion:** this was a true blind spot, not a broken rule. The next step
was writing a detection for it from scratch.

## Step 3 — Validating against live attack traffic

Writing a rule that passes syntax validation isn't the same as confirming it
actually fires for real attack traffic — that was the exact mistake the
original Kerberoasting rule made. So before considering this done, I
generated a fresh AS-REP request against `svc-backup` from the attacker
workstation and checked whether the new rule caught it.

```
sudo grep '"id":"100132"' /var/ossec/logs/alerts/alerts.json
```

<details>
<summary>What this does</summary>

| Part | What it does |
|---|---|
| `grep '"id":"100132"'` | Searches the live alerts log for any alert matching the new rule's ID |

</details>

The alert fired immediately:

```json
{
  "rule": {
    "level": 12,
    "description": "Possible AS-REP Roasting: TGT requested without Kerberos pre-authentication for svc-backup from ::ffff:10.0.50.10",
    "id": "100132",
    "mitre": { "id": ["T1558.004"], "tactic": ["Credential Access"], "technique": ["AS-REP Roasting"] }
  }
}
```

<img width="811" height="273" alt="image" src="https://github.com/user-attachments/assets/b6b53fc3-e9af-4652-92c8-b043a2169a37" />

This confirms three things at once:

1. The rule correctly identifies `svc-backup` as the targeted account.
2. The alert correctly tags the activity under MITRE ATT&CK `T1558.004`
   (AS-REP Roasting).
3. **The alert fired despite the source address being reported in the same
   IPv4-mapped IPv6 notation (`::ffff:10.0.50.10`) that silently defeated the
   Kerberoasting rule.** This validates the root cause identified in the
   Kerberoasting investigation — the problem was never IPv6 traffic itself,
   it was specifically matching a rule `<field>` condition directly against
   that notation. Avoiding that pattern here produced a rule that works
   correctly on the first real attempt, against real remote attacker
   traffic, with no debugging required.

Unlike the Kerberoasting scenario, this gap didn't need a compensating
control or a documented limitation — it got a working fix, validated against
the actual attack it was meant to catch.

## Step 4 — Finding & remediation

### Finding

AS-REP roasting against `svc-backup` had **zero detection coverage** prior to
this investigation — not a broken rule, but a complete absence of one. Event
ID 4768 wasn't covered by any built-in Wazuh rule, meaning an attacker could
have requested and captured an AS-REP hash for this account with no alert of
any kind, regardless of how the request was made or from where.

Unlike the Kerberoasting finding, this gap was fully closed rather than
documented as an open limitation. A new rule (`100132`) was written, anchored
using the sibling-matching lesson from the Kerberoasting investigation, and
deliberately designed to avoid matching directly on the `ipAddress` field —
the specific pattern responsible for the earlier rule's failure on remote
traffic. The rule was then validated against a real, remote AS-REP request
and fired correctly on the first attempt.

This also retroactively strengthens the Kerberoasting finding: the working
behavior of this rule under the same IPv4-mapped IPv6 conditions confirms the
root cause identified there was correct, and shows a concrete way to design
future rules so they aren't affected by it.

### Remediation

- **Detection gap closed.** Rule `100132` now alerts on any Kerberos TGT
  request where pre-authentication was not used, tagged under MITRE ATT&CK
  `T1558.004`.
- **Credential exposure remediated.** `svc-backup`'s password was rotated
  immediately after the attack chain was completed, since the cracked
  password (`Fall2024!!`) was treated as compromised the moment it appeared
  in attack evidence — the same standard applied to any real incident.
- **Underlying misconfiguration should still be corrected.** The rule
  detects exploitation attempts, but it doesn't fix the root cause: 
  `svc-backup` still has Kerberos preauthentication disabled. Preauth should
  be re-enabled for this account unless there's a specific, documented reason
  it can't be (legacy application compatibility is the usual case) — a
  detection is a safety net, not a substitute for closing the actual
  exposure.

### Hardening recommendations

- **Re-enable Kerberos preauthentication** on any account where it isn't
  explicitly required to stay disabled.
- **Use Group Managed Service Accounts (gMSAs)** wherever possible, same
  recommendation as the Kerberoasting scenario — gMSAs use long,
  automatically-rotated passwords that make offline cracking of a captured
  AS-REP effectively infeasible even if preauth is somehow disabled again in
  the future.
- **Enforce strong passwords on any service account that can't use a gMSA.**
  This scenario's password (`Fall2024!!`) was deliberately chosen to be
  crackable for demonstration purposes — a genuinely strong, random password
  would have resisted the hybrid mask attack used here the same way the
  Kerberoasting ticket resisted a flat dictionary attack.
- **Monitor for `Pre-Authentication Type: 0`** specifically, rather than
  relying solely on failed-logon-based detections — AS-REP roasting, like
  Kerberoasting, generates no failed authentication events at all.
