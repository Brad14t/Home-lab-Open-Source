# Scenario 02 — AS-REP Roasting

**MITRE ATT&CK:** T1558.004 (Credential Access — AS-REP Roasting)  
**Target:** `svc-backup` (Active Directory service account)

## Summary

This scenario simulates AS-REP roasting against a domain service account
intentionally configured with Kerberos pre-authentication disabled. Using an
already-compromised standard user account (`jcarter`), the attacker performs
authenticated LDAP enumeration to identify vulnerable accounts, requests an
AS-REP hash for `svc-backup` without ever needing its password, and cracks
that hash offline to recover the account's real password.

On the defensive side, this scenario uncovered a **complete detection gap** —
Wazuh had zero rule coverage for this technique at all, not even baseline
visibility. A new custom detection rule was written, deliberately designed
around a lesson learned in Scenario 01 (avoiding a rule-matching bug tied to
IPv4-mapped IPv6 addresses), and validated successfully against live attack
traffic on the first attempt.

## Files in this scenario

| File | Perspective | Contents |
|---|---|---|
| [`attack-playbook.md`](./attack-playbook.md) | Attacker | Step-by-step reproduction of the attack: recon, AS-REP hash capture, offline cracking — written for someone new to the technique |
| [`blue-team-analysis.md`](./blue-team-analysis.md) | Defender | Investigation into existing detection coverage, root-cause-driven rule design, and live validation against real attack traffic |
| [`detection-dashboard.md`](./detection-dashboard.md) | Defender | Documentation of the Wazuh dashboard panels covering this scenario (part of a combined, running dashboard across all scenarios) |
| [`incident-report.md`](./incident-report.md) | Defender | Screenshots of the full case record built in DFIR-IRIS — summary, notes, assets, IOCs, timeline, graph, and tasks |

## Outcome

- Full attack chain demonstrated end-to-end, including offline password
  recovery.
- Detection gap identified, closed, and proven effective against real traffic
  — unlike Scenario 01, this one ends with a working fix rather than a
  documented limitation.
- Compromised credential rotated as part of remediation.
- The account's root-cause misconfiguration (preauth disabled) is left in
  place intentionally, tracked as an open task in IRIS, with the new
  detection rule serving as the compensating control.
