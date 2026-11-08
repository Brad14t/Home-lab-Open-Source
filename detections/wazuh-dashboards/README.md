<!-- screenshot: full "SOC — Authentication Overview" dashboard -->

# Wazuh Dashboard — SOC — Authentication Overview

A morning-review dashboard for the SOC: within a few seconds it answers *"did any malicious logins happen overnight?"* and *"who logged in, how often, and from where?"*. It's separate from the per-scenario alert dashboards and built entirely on the authentication rule block [`100160`–`100169`](../wazuh-local-rules/README.md#100160100169-authentication-monitoring).

| Setting | Value |
|---|---|
| Index pattern | `wazuh-alerts-*` |
| Default time range | Last 24 hours (stored with the dashboard) |
| Scope | All human accounts on Windows (DC01/02, WS01/02) + all users on Linux (vuln-docker-01) |

## Layout

| Row | Left | Right |
|---|---|---|
| R0 | Header (Markdown): scope + triage guide | Filters (Controls): Host, Severity |
| R1 | Activity Volume (metric) | Threat Indicators (metric, red when > 0) |
| R2 | Logins Over Time (stacked bar) | Logins by Source Zone (horizontal bar) |
| R3 | User Activity by Hour (heatmap) | Logon Types (donut) |
| R4 | Top Users by Failed Logins (bar) | Top Source IPs (bar) |
| R5 | Triage Queue (saved search, full width) | |
| R6 | Login Detail (data table, full width) | |

## Query sets used throughout

| Name | Query |
|---|---|
| **Success** | `rule.id:(100160 or 100164 or 100168 or 100169)` |
| **Failure** | `rule.id:(100161 or 100165)` |
| **All logins** | `rule.id:(100160 or 100161 or 100164 or 100165 or 100168 or 100169)` |

> ATTACKER-VLAN logons are recorded as `100168`/`100169` instead of `100160`/`100164`, which is why every success set lists all four IDs.

---

## Scripted fields

Windows and Linux store the same information in different fields, and Windows adds an `::ffff:` prefix to IPv4 addresses. Two painless scripted fields on `wazuh-alerts-*` merge them, so any panel can aggregate across both platforms.

| Field | Merges | Normalization |
|---|---|---|
| `auth_user` | `data.win.eventdata.targetUserName` / `data.dstuser` | Lowercased |
| `auth_src_ip` | `data.win.eventdata.ipAddress` / `data.srcip` | Strips `::ffff:`; maps `-`, `127.0.0.1` and `::1` to `local` |

To add them: Dashboards Management → Index patterns → `wazuh-alerts-*` → Scripted fields. Language painless, type string.

<details><summary><code>auth_user</code></summary>

```java
if (doc.containsKey('data.win.eventdata.targetUserName') && doc['data.win.eventdata.targetUserName'].size() > 0) { return doc['data.win.eventdata.targetUserName'].value.toLowerCase(); }
if (doc.containsKey('data.dstuser') && doc['data.dstuser'].size() > 0) { return doc['data.dstuser'].value.toLowerCase(); }
return null;
```
</details>

<details><summary><code>auth_src_ip</code></summary>

```java
String ip = null;
if (doc.containsKey('data.win.eventdata.ipAddress') && doc['data.win.eventdata.ipAddress'].size() > 0) { ip = doc['data.win.eventdata.ipAddress'].value; }
else if (doc.containsKey('data.srcip') && doc['data.srcip'].size() > 0) { ip = doc['data.srcip'].value; }
if (ip == null) { return null; }
if (ip.startsWith('::ffff:')) { ip = ip.substring(7); }
if (ip == '-' || ip == '127.0.0.1' || ip == '::1') { return 'local'; }
return ip;
```
</details>

Scripted fields can be used in visualizations and Discover columns, but not in Controls. To filter by user, type `auth_user:<name>` in the query bar.

---

## Panels

### Auth - Header (Markdown)
States the scope and the triage path: any red tile → Triage Queue → raise an IRIS case. A logon from 10.0.50.0/24 (ATTACKER) is malicious by definition.

### Auth - Filters (Controls)

| Control | Type | Field |
|---|---|---|
| Host | Options list | `agent.name` |
| Severity | Range slider | `rule.level` |

Save the dashboard with **no hosts selected**. A saved selection becomes a pinned filter that silently excludes any agent added later.

### Auth - Activity Volume (Metric)
Neutral counts with no color ranges. Query: `rule.groups:lab_accounts`, split group by Filters:

| Tile | Filter |
|---|---|
| Successful logins | **Success** |
| Failed logins | **Failure** |
| Sudo to root | `rule.id:100166` |

### Auth - Threat Indicators (Metric)
Turns red when any value is above 0. Color ranges: `0 → 0.5` green, `0.5 → 100000` red, background mode. Query: `rule.groups:lab_accounts`.

| Tile | Filter |
|---|---|
| Brute force | `rule.id:(100162 or 100167)` |
| Service acct interactive | `rule.id:100163` |
| Logins from ATTACKER VLAN | `rule.id:(100168 or 100169)` |

### Auth - Logins Over Time (Vertical bar, stacked)
Query: `rule.groups:lab_accounts`.
- X-axis: date histogram on `@timestamp`, hourly.
- Split series by Filters: **Success** (green) and Failure = `rule.id:(100161 or 100162 or 100165 or 100167)` (red).
- The grey bar at the right edge is the current, incomplete hour.

### Auth - Logins by Source Zone (Horizontal bar, stacked)
Answers "from where" using the lab's VLANs. With private IPs a geo map isn't meaningful, so zones are. Query: **All logins**.
- X-axis: Filters, one per zone (below).
- Split series: **Success** / **Failure**.

| Zone | Filter |
|---|---|
| ATTACKER | `data.win.eventdata.ipAddress:*10.0.50.* or data.srcip:10.0.50.*` |
| VULNERABLE | `data.win.eventdata.ipAddress:*10.0.40.* or data.srcip:10.0.40.*` |
| CORPORATE | `data.win.eventdata.ipAddress:*10.0.30.* or data.srcip:10.0.30.*` |
| SECURITY | `data.win.eventdata.ipAddress:*10.0.20.* or data.srcip:10.0.20.*` |
| MGMT / Home LAN | `data.win.eventdata.ipAddress:*192.168.0.* or data.srcip:192.168.0.*` |
| Local / no IP | `data.win.eventdata.ipAddress:("-" or "127.0.0.1" or "::1") or (not data.win.eventdata.ipAddress:* and not data.srcip:*)` |

### Auth - User Activity by Hour (Heat map)
Off-hours logons and unexpected accounts (e.g. `svc-*`) stand out visually. Query: **All logins**.
- X-axis: hourly date histogram.
- Y-axis: Terms on `auth_user`, size 15.
- Color schema: Yellow to Red.

### Auth - Logon Types (Donut)
Query: **All logins**. Split slices by Filters:

| Slice | Filter |
|---|---|
| Interactive (2) | `data.win.eventdata.logonType:2` |
| Network (3) | `data.win.eventdata.logonType:3` |
| Unlock (7) | `data.win.eventdata.logonType:7` |
| RDP (10) | `data.win.eventdata.logonType:10` |
| Cached (11) | `data.win.eventdata.logonType:11` |
| Linux SSH / console | `rule.id:(100164 or 100165 or 100169)` |

### Auth - Top Users by Failed Logins (Horizontal bar)
Spots password spraying and brute-force targets. Query: **Failure**. Terms on `auth_user`, size 10, red bars.

### Auth - Top Source IPs (Horizontal bar, stacked)
Query: **All logins**. Terms on `auth_src_ip`, size 10, split series **Success** / **Failure**.

### Auth - Triage Queue (Saved search)
The rows an analyst actually opens in IRIS. Query: `rule.groups:lab_accounts and rule.level >= 10`.

Columns:

| Column |
|---|
| `agent.name` |
| `auth_user` |
| `auth_src_ip` |
| `rule.id` |
| `rule.level` |
| `rule.description` |

### Auth - Login Detail (Data table)
Drill-down evidence. Query: **All logins**. Split rows: `auth_user` → `agent.name` → `auth_src_ip` → `rule.description` (Terms, size 20 each, partial rows shown).

---

## What it currently shows

Captured 2026-11-08, the day the dashboard was built. All activity is from rule validation testing.

| Panel | Value | Meaning |
|---|---|---|
| Successful logins | 76 | Mostly `jcarter` network logons from WS01 (10.0.30.102) to the DCs: Group Policy and SYSVOL after each sign-in |
| Failed logins | 9 | `docker_user01` 5 (SSH tests), `jcarter` 4 (lock-screen tests) |
| Sudo to root | 3 | `sudo` tests on vuln-docker-01 |
| Brute force | 2 🔴 | One validation test each for 100162 (Windows) and 100167 (Linux) |
| Service acct interactive | 1 🔴 | `runas /user:lab\svc-sql` validation test for 100163 |
| Logins from ATTACKER VLAN | 1 🔴 | SSH from Kali (10.0.50.10) to vuln-docker-01, the 100169 validation test |
| Logon types | Network ~74%, Linux ~9%, Cached ~6%, Unlock ~6%, Interactive ~5% | Typical domain workstation mix |

## Known gaps

| Gap | Impact |
|---|---|
| Rule `100168` (Windows logon from ATTACKER) not yet tested | ATTACKER → CORPORATE is blocked by pfSense. To be validated in Scenario 04 with the temporary ATTACKER → DC rule |
| DC02 and WS02 only report while powered on | RAM limits mean not every VM runs at once. Their absence isn't evidence of no logons |
| Splunk (102), IRIS (103/104) and Metasploitable (131) have no agents | Logons to the SOC tooling itself aren't monitored |
| Panel history starts 2026-11-08 | The rules didn't exist before then; the 24-hour charts fill as data accumulates |
