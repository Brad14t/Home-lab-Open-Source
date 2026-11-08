# Detections

Custom detection content built for this lab. Every rule here was validated against live traffic in the lab; where a rule has a known blind spot, that gap is documented rather than hidden.

| Folder | Contents |
|---|---|
| [wazuh-local-rules/](wazuh-local-rules/) | All custom Wazuh rules (`100100`–`100169`) and decoders: ID, level, MITRE mapping, log source, parent rules, validation status and known limitations |

## Rule ID blocks

Rules are allocated in blocks of 10 per detection area, so related rules stay together and the next free ID is always obvious.

| Block | Detection area | Built in |
|---|---|---|
| `100100`–`100103` | Noise tuning (dpkg, rootcheck, SCA, FIM) | Phase 6 |
| `100110`–`100112` | pfSense Suricata IDS alerts | Phase 6 |
| `100120`–`100122` | pfSense firewall blocks (filterlog) | Phase 6 |
| `100130`–`100132` | Kerberos credential attacks | [Scenario 01](../scenarios/01-kerberoasting/), [Scenario 02](../scenarios/02-asrep-roasting/) |
| `100140`–`100141` | Portainer brute force | [Scenario 03](../scenarios/03-web-attacks/command-injection/) |
| `100150`–`100151` | DVWA command injection | [Scenario 03](../scenarios/03-web-attacks/command-injection/) |
| `100160`–`100169` | Authentication monitoring (Windows + Linux logons, failures, brute force, sudo, logons from the ATTACKER VLAN) | SOC Authentication Overview dashboard |
| `100170`+ | **Next free block:** sensitive file access (planned) | — |

## Dashboards

Scenario-specific dashboards are documented inside each scenario folder (`detection-dashboard.md`). The cross-scenario **SOC — Authentication Overview** dashboard is built on rules `100160`–`100169`; its documentation will be added under `detections/` together with the planned Sensitive File Access dashboard.

## Conventions

| Convention | Why |
|---|---|
| Tune by rule, never by raising the global alert threshold | Keeps every other detection intact |
| Validate (`wazuh-analysisd -t`) before every restart | A bad rule file stops the manager |
| Back up `local_rules.xml` before every change (`local_rules.xml.bak-<date>`) | One-command rollback |
| Prove every rule on live traffic, not only `wazuh-logtest` | Test events and real events can differ (see Scenario 01's `::ffff:` finding) |
