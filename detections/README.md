# Detections

Custom detection content built for this lab. Every rule here was validated against live traffic in the lab; where a rule has a known blind spot, that gap is documented rather than hidden.

| Folder | Contents |
|---|---|
| [wazuh-dashboards/](wazuh-dashboards/) | Cross-scenario SOC dashboards: [SOC — Authentication Overview](wazuh-dashboards/authentication-overview.md) and [SOC — Sensitive File Access](wazuh-dashboards/sensitive-file-access.md) (panels, queries, scripted fields); see the [folder README](wazuh-dashboards/README.md) for shared conventions |
| [wazuh-local-rules/](wazuh-local-rules/) | All custom Wazuh rules (`100100`–`100177`) and decoders: ID, level, MITRE mapping, log source, parent rules, validation status and known limitations |

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
| `100170`–`100179` | Sensitive file access and credential dumping (`100170`–`100177` built, `100178` planned for Windows 4663 file reads) | SOC Sensitive File Access dashboard |
| `100180`+ | **Next free block** | — |

## Dashboards

| Dashboard | Built on | Doc |
|---|---|---|
| Scenario detection panels | One block per scenario | `detection-dashboard.md` in each [scenario folder](../scenarios/) |
| SOC — Authentication Overview | `100160`–`100169` | [wazuh-dashboards/authentication-overview.md](wazuh-dashboards/authentication-overview.md) |
| SOC — Sensitive File Access | `100170`–`100177` + built-ins `92026`, `92900` | [wazuh-dashboards/sensitive-file-access.md](wazuh-dashboards/sensitive-file-access.md) |

## Conventions

| Convention | Why |
|---|---|
| Tune by rule, never by raising the global alert threshold | Keeps every other detection intact |
| Validate (`wazuh-analysisd -t`) before every restart | A bad rule file stops the manager |
| Back up `local_rules.xml` before every change (`local_rules.xml.bak-<date>`) | One-command rollback |
| Prove every rule on live traffic, not only `wazuh-logtest` | Test events and real events can differ (see Scenario 01's `::ffff:` finding) |
