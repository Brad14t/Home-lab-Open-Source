<img width="955" height="303" alt="image" src="https://github.com/user-attachments/assets/bae779b3-236b-4058-920c-3fc83cdb0d33" />

# Wazuh Dashboard — Kerberoasting Detection

A two-panel Wazuh dashboard built on the custom Kerberoasting rules (`100130`,
`100131`) Built to give a quick, at-a-glance view of Kerberoasting-related alerts without having to
hand-write a Discover query every time.

## Panels

### Kerberoasting - Alert Count
A single metric panel showing the total count of alerts matched by rules
`100130` and `100131` over the selected time range. A quick top-level signal —
if this is `0`, no Kerberoasting-pattern activity has triggered a detection;
anything above `0` means it's time to open the table panel and investigate.

### Kerberoasting - Targets & Users
A data table breaking down every alert by:

| Column | Shows |
|---|---|
| `serviceName` | Which service account was targeted (e.g. `svc-sql`, `svc-backup`) |
| `targetUserName` | Which account actually requested the ticket |
| `Count` | How many times that specific pairing has fired |

This is the panel that matters for triage — it immediately answers *who
requested a ticket for which service account, and how often*, without
digging through raw events one at a time.

## What it currently shows

As of this writing, the dashboard shows exactly **2 alerts**, both for
`svc-sql` requested by `adm-brad@LAB.LOCAL`. This is not a false reading —
it's an accurate reflection of a known detection gap documented in
[`blue-team-analysis.md`](./blue-team-analysis.md): those two alerts are from
local/loopback rule testing, not the real Kerberoasting attack carried out in
this scenario. The real attack (`jcarter@LAB.LOCAL` requesting `svc-sql` from
a remote workstation) does **not** appear here, because the underlying rule
doesn't currently fire for remote-sourced events.

This dashboard is left in place intentionally, gap and all — it's an honest
view of what the detection pipeline actually catches right now, not a
cleaned-up version that hides the problem.
