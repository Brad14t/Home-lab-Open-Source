<img width="946" height="469" alt="image" src="https://github.com/user-attachments/assets/96f4c96d-bf20-4ee0-a570-70d35938b6cb" />

# Wazuh Dashboard — Detection Panels (AS-REP Roasting addition)

This dashboard is a **running, combined view** across all detection rules
built so far in this lab — it started with the Kerberoasting panels in
[Scenario 01](../01-kerberoasting/detection-dashboard.md) and now includes
AS-REP roasting detection (`100132`) from this scenario. Rather than a
separate dashboard per attack, detections are added here as each scenario is
completed, so this single view grows into a full picture of what's actually
being monitored across the lab.

## Panels

### Kerberoasting - Alert Count / Kerberoasting - Targets & Users
Covered in [Scenario 01's dashboard doc](../01-kerberoasting/detection-dashboard.md).
Unchanged here — included for the combined view.

### AS-REP Roasting - Alert Count
A metric panel showing the total count of alerts matched by rule `100132`
(AS-REP roasting — TGT requested with Kerberos pre-authentication disabled)
over the selected time range.

### AS-REP Roasting - Targets
A data table breaking down every AS-REP roasting alert by:

| Column | Shows |
|---|---|
| `targetUserName` | Which account's TGT was requested without pre-authentication (e.g. `svc-backup`) |
| `ipAddress` | Where the request originated from |
| `Count` | How many times that specific pairing has fired |

Note this table splits on `targetUserName` rather than `serviceName` like the
Kerberoasting table does — AS-REP requests always show `serviceName: krbtgt`,
so the account actually being targeted is the *requester*, not the service
field.

## What it currently shows

As of this writing, the AS-REP panels show **1 alert**, for `svc-backup`
requested from `::ffff:10.0.50.10` — the real attack traffic from this
scenario's `attack-playbook.md`, successfully caught by rule `100132` on the
first live attempt. Unlike the Kerberoasting panels (which only ever show the
local test-fire alerts due to the documented detection gap in
[Scenario 01's blue-team-analysis.md](../01-kerberoasting/blue-team-analysis.md)),
this panel reflects a real, correctly-detected attack — a working example of
the detection pipeline doing exactly what it's supposed to.
