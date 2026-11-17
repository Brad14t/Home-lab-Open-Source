<!-- screenshot: full "SOC — Sensitive File Access" dashboard -->

# Wazuh Dashboard — SOC — Sensitive File Access

A morning-review dashboard for the SOC: within a few seconds it answers *"did anyone read or change a credential store, dump credentials, or escape a container overnight?"*. It's separate from the per-scenario alert dashboards and is built on the sensitive-file rule block [`100170`–`100177`](../wazuh-local-rules/README.md#100170100177-sensitive-file-access-and-credential-dumping) plus two built-in Wazuh rules (`92026`, `92900`).

| Setting | Value |
|---|---|
| Index pattern | `wazuh-alerts-*` |
| Default time range | Last 24 hours (stored with the dashboard) |
| Scope | Linux: vuln-docker-01 (auditd). Windows: DC01/02, WS01/02 (Sysmon event 1 and 10, Security 4688) |

## Layout

| Row | Panels |
|---|---|
| R0 | Header (Markdown, full width): scope + triage guide |
| R1 | Filters (Controls: Host, Severity) · Activity Volume (metric) · Threat Indicators (metric, red when > 0) |
| R2 | Events Over Time (stacked bar) · By Technique (donut) |
| R3 | Top Hosts (bar) · Top Processes (bar) · Top Users (bar) |
| R4 | Triage Queue (saved search, full width) |
| R5 | Detail (saved search, full width) |
| R6 | Coverage & Gaps (Markdown, full width) |

## Query sets used throughout

| Name | Query |
|---|---|
| **Linux** | `rule.id:(100170 or 100171 or 100172 or 100173)` |
| **Windows** | `rule.id:(92026 or 92900 or 100174 or 100175 or 100176 or 100177)` |
| **ALL** | `rule.id:(92026 or 92900 or 100170 or 100171 or 100172 or 100173 or 100174 or 100175 or 100176 or 100177)` |

> The queries select by rule ID, not by group. The built-in rules `92026` (`reg save` of the SAM hive) and `92900` (lsass access) cover techniques this dashboard needs, but they aren't in the `sensitive_file` group that the custom rules share.

> **Every visualization must carry the ALL query itself.** A panel built without a saved query silently counts every alert in the index. This showed up on build day: five panels were charting all alerts (Top Hosts totalled thousands against an Activity Volume of 411) until the query was saved on each one.

---

## Scripted fields

Windows and Linux log the acting process and user in different fields. Two painless scripted fields on `wazuh-alerts-*` merge them, so one panel can aggregate across both platforms.

| Field | Merges | Normalization |
|---|---|---|
| `sens_process` | `data.win.eventdata.image` / `sourceImage` / `newProcessName` (Sysmon 1, Sysmon 10, Security 4688) / `data.audit.exe` (auditd) | Basename only, lowercased (`reg.exe`, `systemd`) |
| `sens_user` | `data.win.eventdata.user` / `subjectUserName` / `data.audit.auid` | Strips the `DOMAIN\` prefix, lowercased. Linux `auid` shows as `auid 1000`; `4294967295` (no login user, i.e. a daemon) shows as `unset (system)` |

To add them: Dashboards Management → Index patterns → `wazuh-alerts-*` → Scripted fields. Language painless, type string.

> **Two painless pitfalls hit while building these.** `String[] f = [...]` does not compile (use a `List` or flat `if` chains), and `Math.max` failed in the scripted-field context, so the script compares the two `lastIndexOf` results with a ternary instead. Also, OpenSearch limits dynamic script compilation to **75 per 5 minutes**, and the scripted-field editor recompiles on every edit. After a few failed attempts every save fails with a circuit-breaker error even for correct code. Write the script in a text editor, wait five minutes, paste it once.

<details><summary><code>sens_process</code></summary>

```java
String v = '';
if (doc.containsKey('data.win.eventdata.image') && doc['data.win.eventdata.image'].size() > 0) { v = doc['data.win.eventdata.image'].value; }
else if (doc.containsKey('data.win.eventdata.sourceImage') && doc['data.win.eventdata.sourceImage'].size() > 0) { v = doc['data.win.eventdata.sourceImage'].value; }
else if (doc.containsKey('data.win.eventdata.newProcessName') && doc['data.win.eventdata.newProcessName'].size() > 0) { v = doc['data.win.eventdata.newProcessName'].value; }
else if (doc.containsKey('data.audit.exe') && doc['data.audit.exe'].size() > 0) { v = doc['data.audit.exe'].value; }
if (v == '') { return 'unknown'; }
int a = v.lastIndexOf('\\');
int b = v.lastIndexOf('/');
int i = a > b ? a : b;
String r = i >= 0 ? v.substring(i + 1) : v;
return r.toLowerCase();
```
</details>

<details><summary><code>sens_user</code></summary>

```java
String v = '';
if (doc.containsKey('data.win.eventdata.user') && doc['data.win.eventdata.user'].size() > 0) { v = doc['data.win.eventdata.user'].value; }
else if (doc.containsKey('data.win.eventdata.subjectUserName') && doc['data.win.eventdata.subjectUserName'].size() > 0) { v = doc['data.win.eventdata.subjectUserName'].value; }
if (v != '') {
  int i = v.lastIndexOf('\\');
  String r = i >= 0 ? v.substring(i + 1) : v;
  return r.toLowerCase();
}
if (doc.containsKey('data.audit.auid') && doc['data.audit.auid'].size() > 0) {
  String a = String.valueOf(doc['data.audit.auid'].value);
  return a == '4294967295' ? 'unset (system)' : 'auid ' + a;
}
return 'unknown';
```
</details>

Scripted fields can be used in visualizations and Discover columns, but not in Controls.

---

## Panels

### SOC-SFA — Header (Markdown)
States the scope, the rule IDs covered, and that red means level 12 or higher.

### SOC-SFA — Filters (Controls)

| Control | Type | Field |
|---|---|---|
| Host | Options list | `agent.name` |
| Severity | Options list | `rule.level` |

Save the dashboard with **nothing selected**. A saved selection becomes a pinned filter that silently excludes any agent added later (troubleshooting #31).

### SOC-SFA — Activity Volume (Metric)
Query: **ALL**. Count, labelled *Sensitive-file & credential events*. No color ranges.

### SOC-SFA — Threat Indicators (Metric)
Query: **ALL**. Count, split group by Filters. Turns red when any value is above 0.

| Tile | Filter |
|---|---|
| Credential store read | `rule.id:100170` |
| Persistence / priv change | `rule.id:100171` |
| Container escape | `rule.id:100173` |
| Windows credential dump | `rule.id:(92026 or 92900 or 100174 or 100175 or 100176 or 100177)` |

Color ranges: `0 → 0.5` green, `0.5 → 100000` red. A `0 → 1` range would leave a tile showing exactly 1 green (troubleshooting #30).

### SOC-SFA — Events Over Time (Vertical bar, stacked)
Query: **ALL**. X-axis: date histogram on `@timestamp` (auto interval). Split series: Terms on `rule.id`, top 10 by Count. Shows when activity happened and which rule drove it.

### SOC-SFA — By Technique (Donut)
Query: **ALL**. Split slices: Terms on `rule.mitre.technique`, size 10.

### SOC-SFA — Top Hosts (Horizontal bar)
Query: **ALL**. Terms on `agent.name`, size 10.

### SOC-SFA — Top Processes (Horizontal bar)
Query: **ALL**. Terms on `sens_process`, size 10. Typical values: `reg.exe`, `vssadmin.exe`, `esentutl.exe`, `ntdsutil.exe`, `cat`, `mount`.

### SOC-SFA — Top Users (Horizontal bar)
Query: **ALL**. Terms on `sens_user`, size 10. Shows `unknown` when an alert carries none of the user fields `sens_user` checks. That has not been investigated further.

### SOC-SFA — Triage Queue (Saved search)
The rows an analyst opens first. Query: **ALL** `and rule.level >= 10`. Sorted by `@timestamp`, newest first.

| Column |
|---|
| `agent.name` |
| `rule.id` |
| `rule.level` |
| `rule.description` |
| `sens_process` |

> **Saved-search overwrite.** Opening a saved search in Discover and then saving it, even under a new title, overwrites the original. While building this dashboard, the authentication dashboard's `Auth - Triage Queue` search was overwritten with this dashboard's query. Always click **New** in Discover before building a saved search, and check the other dashboard afterwards.

### SOC-SFA — Detail (Saved search)
Drill-down evidence. Query: **ALL**. Sorted by `@timestamp`, newest first.

| Column |
|---|
| `agent.name` |
| `rule.id` |
| `rule.level` |
| `rule.description` |
| `sens_process` |
| `data.audit.file.name` |
| `data.win.eventdata.commandLine` |

### SOC-SFA — Coverage & Gaps (Markdown)
States what is and isn't covered, so the dashboard never implies coverage it doesn't have. See *Known gaps* below.

---

## What it currently shows

Captured 2026-11-15, Last 24 hours. All activity is from rule validation testing.

| Panel | Value | Meaning |
|---|---|---|
| Activity Volume | 411 | Equals the sum of the four Threat Indicators tiles |
| Credential store read | 7 🔴 | 4 from a Wazuh binary (`/var/ossec/bin/wazuh*`) and 2 from systemd (all before 100170 was tuned), plus 1 deliberate `cat /etc/shadow` test |
| Persistence / priv change | 0 🟢 | Nothing touched sudoers, authorized_keys or cron |
| Container escape | 395 🔴 | 392 are systemd service-sandbox mounts, logged before 100173 was tuned. The other 3 are `/usr/bin/mount` hits, including a deliberate test and an automated loop-device mount (auid unset, likely snapd) |
| Windows credential dump | 9 🔴 | All deliberate validation tests (92026, 100174, 100175, 100176) |

The systemd noise was cut from hundreds of alerts to none by the tuning described in the [rules README](../wazuh-local-rules/README.md#100170100177-sensitive-file-access-and-credential-dumping). The pre-tuning alerts stay in the index until they age out of the 24-hour range.

## Known gaps

| Gap | Impact |
|---|---|
| **Windows file reads (event 4663) are not covered** | Opening `SAM`, `NTDS.dit` or other sensitive files is invisible. File System auditing is set to "No Auditing", the agent's `<query>` excludes 4663, and no SACLs exist. Planned as rule `100178` |
| lsass access (`100177`, built-in `92900`) is unvalidated on live traffic | Sysmon never logged a benign lsass handle, and `wazuh-logtest` can't validate EventChannel rules. To be validated in Scenario 04 with a real dump technique |
| `ntdsutil` is not logged by Sysmon (sysmon-modular) | Covered only through Security 4688 (`100175`). A host without command-line auditing on 4688 would miss it |
| No DCSync rule (event 4662) | Directory replication abuse is not detected |
| Command-based rules match tool names | A renamed binary, `comsvcs.dll` MiniDump, or shadow copies created through WMI or PowerShell evade `100174`–`100176` |
| Linux coverage is vuln-docker-01 only | It is the only Linux host with an agent. Metasploitable (131) has none |
| systemd and Wazuh-agent exclusions | `100170`/`100173` ignore `/usr/lib/systemd/systemd` and `/var/ossec/bin/wazuh*`. A malicious systemd unit, or a process masquerading as the agent binary (which needs root), would be missed |
| Loop-device mounts by `/usr/bin/mount` with auid unset | Probably snapd; still alerts at level 12 and is a candidate for tuning |
