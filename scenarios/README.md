# Scenarios

Each scenario runs a real attack technique against the lab, then verifies and
documents the detection. All follow the same structure — see [_TEMPLATE.md](_TEMPLATE.md).

| # | Scenario | MITRE ATT&CK | Status |
|---|---|---|---|
| 01 | [Kerberoasting](01-kerberoasting/) | T1558.003 | ✅ Complete |
| 02 | [AS-REP Roasting](02-asrep-roasting/) | T1558.004 | ✅ Complete |
| 03 | [DCSync](03-dcsync/) | T1003.006 | ⬜ Planned |
| 04 | [Web Attacks / Command Injection](04-web-attacks/command-injection/) | T1190, T1059.004, T1110, T1611, T1003.008 | ✅ Complete |
| 05 | [Caldera Emulation](05-caldera-emulation/) | Multiple | ⬜ Planned |

## Files in each scenario

Every scenario folder contains the same five artifacts, each written from a
distinct perspective:

| File | Perspective | Contents |
|---|---|---|
| `attack-playbook.md` | Attacker (red team) | Step-by-step offensive actions, real commands and output, command-explanation tables |
| `blue-team-analysis.md` | Defender (blue team) | Full investigative narrative, written with no foreknowledge of the attack — hunt methodology, real Discover queries, what fired, what didn't, and why |
| `detection-dashboard.md` | Defender (blue team) | Wazuh dashboard panels built for the scenario's detections — what each panel shows and what it currently displays, gaps included |
| `IRIS-case-<scenario-name>.md` | Both | DFIR-IRIS case record — Summary, Notes, Assets, IOCs, Timeline, Tasks, and Evidence |
| `incident-report/` | Defender (blue team) | Screenshots of the DFIR-IRIS case tabs (Summary, Assets, IOC, Timeline, Tasks, Graph, Evidence) |

The blue team documents are written as a genuine investigation, not a
retroactive summary — the analyst has no access to the attack-playbook's
content and works entirely from what the detection pipeline and raw logs
actually show.
