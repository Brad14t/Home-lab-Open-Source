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

Once a fresh AS-REP request was generated, this confirmed the event *was*
being ingested and fully decoded by Wazuh (all fields present and correct) —
it just had zero rule coverage, so it never reached the alerts index that
Discover searches by default.

**Conclusion:** this was a true blind spot, not a broken rule. The next step
was writing a detection for it from scratch.
