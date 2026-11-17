# Wazuh Dashboards

Cross-scenario SOC dashboards, built on the Wazuh dashboard (OpenSearch Dashboards) against `wazuh-alerts-*`. They are separate from the per-scenario alert panels in each [scenario folder](../../scenarios/): those prove one attack was detected, these are the **morning-review** views an analyst opens to see what happened overnight.

| Dashboard | Question it answers | Built on | Doc |
|---|---|---|---|
| SOC — Authentication Overview | Who logged in, from where, how often, and did any login look malicious? | Rules [`100160`–`100169`](../wazuh-local-rules/README.md#100160100169-authentication-monitoring) | [authentication-overview.md](authentication-overview.md) |
| SOC — Sensitive File Access | Did anyone read or change a credential store, dump credentials, or escape a container? | Rules [`100170`–`100177`](../wazuh-local-rules/README.md#100170100177-sensitive-file-access-and-credential-dumping) + built-ins `92026`, `92900` | [sensitive-file-access.md](sensitive-file-access.md) |

Both share the same layout and triage flow: a header and filters, two metric tiles (activity volume, and threat indicators that turn red above 0), trend and breakdown charts, a **Triage Queue** of level 10+ alerts to open in IRIS, and a full detail table. Each lists its known gaps explicitly, so a quiet dashboard is never mistaken for full coverage.

## Shared conventions

| Convention | Why |
|---|---|
| Index pattern `wazuh-alerts-*`, default range Last 24 hours stored with the dashboard | Same morning-review window on every dashboard |
| Queries select by **rule ID**, and each visualization saves its own query | Built-in rules (e.g. `92026`) aren't in the custom groups, and a panel with no saved query silently counts every alert (it happened on build day) |
| Threat indicator color ranges `0 → 0.5` green, `0.5 → 100000` red | A `0 → 1` range leaves a tile showing exactly 1 green (troubleshooting #30) |
| Save the Controls panel with **nothing selected** | A saved selection becomes a pinned filter that silently excludes later-added agents (troubleshooting #31) |
| Scripted fields merge Windows and Linux fields so one panel covers both | Authentication: `auth_user`, `auth_src_ip`. Sensitive File Access: `sens_process`, `sens_user`. Scripted fields work in visualizations and Discover but not in Controls |
| Click **New** in Discover before building a saved search | Saving a loaded saved search overwrites it, even under a new title (troubleshooting #50) |
| Panel names are prefixed per dashboard (`Auth - …`, `SOC-SFA — …`) | Saved objects share one namespace, so the prefix shows which dashboard owns a panel |

## Rebuilding a dashboard

1. Rules first: the dashboards only read alerts, so the rule block must be loaded and validated ([rules README](../wazuh-local-rules/README.md)).
2. Add the scripted fields on `wazuh-alerts-*` (code in each dashboard doc). Mind the compile limit: paste once, then save.
3. Build each visualization from the doc's panel section and **save the query on the panel itself**.
4. Build the saved searches from a fresh **New** Discover tab.
5. Assemble the dashboard, set the time range to Last 24 hours, clear the Controls, and save with *Store time with dashboard* on.
6. Reconcile the numbers: the activity tile should equal the sum of the indicator tiles and of the time-series bars.

## Querying scripted fields

Scripted fields can be used in the query bar (for example `auth_user:jcarter`), in visualizations, and as Discover columns. They can't back a Controls panel.
