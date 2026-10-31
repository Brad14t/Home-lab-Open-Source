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
