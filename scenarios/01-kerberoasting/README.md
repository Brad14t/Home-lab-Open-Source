# Scenario 01 — Kerberoasting

**MITRE ATT&CK:** T1558.003 (Credential Access — Kerberoasting)
**Target:** `svc-sql` (Active Directory service account)

## Summary

This scenario simulates Kerberoasting against a domain service account with a
registered Service Principal Name (SPN). Using an already-compromised
standard user account (`jcarter`), the attacker enumerates SPNs domain-wide,
identifies `svc-sql` as Kerberoastable, and requests its service ticket using
a living-off-the-land technique (the built-in `klist get` command) — avoiding
the need for external attacker tooling, which this lab's Windows Server 2025
RC4 hardening would otherwise block. The resulting ticket uses AES256
encryption rather than RC4, reflecting that hardening and making offline
cracking significantly harder than the classic Kerberoasting scenario most
tooling assumes.

On the defensive side, this scenario surfaced a genuine, unresolved detection
gap: a Wazuh rule existed for this exact pattern, but investigation revealed
it only fires reliably for local/loopback-sourced test traffic — not for the
real, remote attacker traffic it was written to catch. Root-cause analysis
traced this to a likely Wazuh-internal issue with IPv4-mapped IPv6 address
notation when that field is used directly as a rule match condition. Rather
than continue indefinite debugging, this was documented as a confirmed
finding, with proactive threat hunting established as the compensating
control. The same root cause was later confirmed and successfully designed
around in [Scenario 02 (AS-REP roasting)](../02-asrep-roasting/README.md).

## Files in this scenario

| File | Perspective | Contents |
|---|---|---|
| [`attack-playbook.md`](./attack-playbook.md) | Attacker | Step-by-step reproduction of the attack: SPN enumeration, living-off-the-land ticket request — written for someone new to the technique |
| [`blue-team-analysis.md`](./blue-team-analysis.md) | Defender | Proactive threat hunt, discovery of the missing alert, and the full detection-engineering investigation into why the existing rule didn't fire |
| [`detection-dashboard.md`](./detection-dashboard.md) | Defender | Documentation of the Wazuh dashboard panels covering this scenario (part of a combined, running dashboard across all scenarios) |
| [`incident-report.md`](./incident-report.md) | Defender | Screenshots of the full case record built in DFIR-IRIS (built retroactively) — summary, notes, assets, IOCs, timeline, graph, and tasks |

## Outcome

- Full attack chain demonstrated end-to-end, including a realistic look at
  how modern AES-hardened Kerberoasting differs from the classic RC4 scenario.
- A real, unresolved detection gap was found, root-caused, and documented
  honestly rather than hidden — the rule fires in testing but not against
  real remote traffic.
- Compensating control (proactive hunting for Kerberos service ticket
  requests against monitored service accounts) is the active mitigation
  pending a rule redesign.
- This finding directly informed a successful fix in the following scenario,
  demonstrating the value of documenting gaps rather than quietly working
  around them.
