# Scenarios

Each scenario runs a real attack technique against the lab, then verifies and
documents the detection. All follow the same structure — see [Files in each scenario](#files-in-each-scenario) below.

| # | Scenario | MITRE ATT&CK | Status (write-up · detection) |
|---|---|---|---|
| 01 | [Kerberoasting](01-kerberoasting/) | T1558.003 | ✅ Complete · ⚠️ detection gap documented (hunting is the compensating control) |
| 02 | [AS-REP Roasting](02-asrep-roasting/) | T1558.004 | ✅ Complete · ✅ detected on live traffic |
| 03 | [Command Injection → Docker Container Escape](03-web-attacks/command-injection/) | T1190, T1059.004, T1110, T1611, T1003.008 | ✅ Complete · ⚠️ partially detected (blind spots documented) |

## Files in each scenario

Every scenario folder contains the same four artifacts (plus a README), each
written from a distinct perspective:

| File | Perspective | Contents |
|---|---|---|
| `attack-playbook.md` | Attacker (red team) | Step-by-step offensive actions, real commands and output, command-explanation tables |
| `blue-team-analysis.md` | Defender (blue team) | Full investigative narrative, written with no foreknowledge of the attack — hunt methodology, real Discover queries, what fired, what didn't, and why |
| `detection-dashboard.md` | Defender (blue team) | Wazuh dashboard panels built for the scenario's detections — what each panel shows and what it currently displays, gaps included |
| `incident-report.md` | Defender (blue team) | Screenshots of the full DFIR-IRIS case record — Summary, Notes, Assets, IOCs, Timeline, Graph, and Tasks |

The blue team documents are written as a genuine investigation, not a
retroactive summary — the analyst has no access to the attack-playbook's
content and works entirely from what the detection pipeline and raw logs
actually show.
