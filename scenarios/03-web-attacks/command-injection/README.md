# Scenario 03 — Command Injection / Docker Escape

**MITRE ATT&CK:** T1190, T1059.004, T1110, T1611, T1003.008 | **Target:** DVWA on vuln-docker-01 (10.0.40.10)

## Summary

An unsanitized input field in DVWA's Command Injection module (CWE-78) was
used to gain a reverse shell on `vuln-docker-01`. From there, Portainer
CE's management API was brute-forced and used to create a privileged
container with the host root filesystem bind-mounted, exposing
`/etc/shadow`. Detection coverage is partial: two Wazuh rules correctly
fire on real attack traffic, but a third of the attack chain — including
the actual exploitation step and a confirmed blind spot in the brute-force
detection — has no coverage at all.

## Files in this scenario

| File | Perspective | Contents |
|---|---|---|
| `attack-playbook.md` | Attacker | Full command-injection chain: endpoint discovery, reverse shell, enumeration, Portainer brute force, privileged container escape, `/etc/shadow` read |
| `blue-team-analysis.md` | Defender | Proactive hunt and detection validation, written with no foreknowledge of the attack. Covers what's detected, what isn't, and the discovery of a structural blind spot in the Portainer brute-force rule |
| `detection-dashboard.md` | Defender | Wazuh dashboard panels built for this scenario's two working detections (`100140`/`100141`, `100150`/`100151`), what each panel shows, and current alert counts |
| [`IRIS-case-command-injection.md`](./IRIS-case-command-injection.md) | Both | Screenshots of the DFIR-IRIS case record — Summary, Notes, Assets, IOCs, Timeline, Tasks, and Evidence for the full investigation |

## Outcome

**Detected:**
- DVWA command injection endpoint activity — rule `100151` (T1190), fires on every POST to `/vulnerabilities/exec/`, but cannot distinguish a benign test from a successful exploit since Apache's log format doesn't capture POST bodies
- Portainer brute force — rule `100141` (T1110), confirmed working **only** when the attacking traffic crosses the pfSense perimeter (external attacker)

**Not detected:**
- Reverse shell establishment (`/dev/tcp`) — no log source captures this at all
- Post-exploitation enumeration inside the container — no process-level visibility on the host
- Portainer brute force run from inside the already-compromised container — the **actual attack path used in this scenario**. Confirmed via live re-test: identical traffic pattern, zero alert, zero raw firewall log entry, because it never leaves the Docker host's internal bridge network
- Privileged container creation and the `/etc/shadow` read — no rule exists; the only corroborating evidence is a Docker daemon lifecycle event recovered through manual `journald` hunting, not an alert

**Key finding:** the Portainer brute-force rule was validated and confirmed working during testing, but that testing only covered an external-attacker scenario. The real attack path in this scenario — brute-forcing a service from a foothold already on the same host — is structurally invisible to a detection built entirely on perimeter firewall logs. A host-based mitigation (`auditd`) was attempted to close this gap and found incomplete (unscoped, process-attribution issues); it's documented as open follow-up work rather than a closed gap.

Full investigative detail, real queries, and screenshots are in
`blue-team-analysis.md` and the DFIR-IRIS case.
