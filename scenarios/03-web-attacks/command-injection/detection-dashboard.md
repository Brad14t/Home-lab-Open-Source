# Detection Dashboard — Command Injection → Docker Container Escape

A four-panel Wazuh dashboard covering both detections built for this
scenario: the Portainer brute-force correlation rule (`100140`/`100141`)
and the DVWA command-injection endpoint rule (`100150`/`100151`). Built to
give a quick, at-a-glance view of activity for both attack phases without
hand-writing a Discover query every time.

## Panels

<img width="952" height="365" alt="Screenshot 2026-11-07 111941" src="https://github.com/user-attachments/assets/0ea1278e-7c5b-47fb-b1bb-da1d1dfe9b3b" />

### Portainer Brute Force - Alert Count

A single metric panel showing the total count of alerts matched by rule
`100141` over the selected time range. `100141` is a frequency-correlation
rule — it only fires once 5 or more connection attempts to Portainer
(`10.0.40.10:9443`) are observed from the same source within a 60-second
window. A quick top-level signal — `0` means no brute-force-pattern traffic
has been observed; anything above `0` means it's time to check the table
panel.

### Portainer Brute Force - Source & Targets

A data table breaking down every `100141` alert by:

| Column | Shows |
|---|---|
| Source IP | `data.srcip` — where the connection attempts originated |
| Destination IP | `data.dstip` — the Portainer host being targeted |
| Count | How many times that source/destination pairing fired |

This is the triage panel — it immediately answers *who is hammering
Portainer, and how many times*, without digging through raw firewall log
events one at a time.

### DVWA Command Injection - Alert Count

A single metric panel showing the total count of alerts matched by rule
`100151` over the selected time range. `100151` fires on any `POST` request
to DVWA's `/vulnerabilities/exec/` endpoint — the command injection
module. A quick top-level signal — `0` means the endpoint hasn't received
a form submission; anything above `0` means someone actually submitted the
form, not just viewed the page.

### DVWA Command Injection - Recent Requests

Unlike the other three panels, this one is a **saved search** rather than
a grouped data table. The raw `data.log` field (the full Apache combined
log line Wazuh extracts from DVWA's container log) isn't mapped as an
aggregatable field in this index, so it can't be bucketed into a Terms
table the way `srcip`/`dstip` can for the Portainer panel. Instead, this
panel lists matching alerts directly:

| Column | Shows |
|---|---|
| Time | When the alert fired |
| `data.log` | The full raw Apache access-log line for the request |
| `rule.description` | The rule's alert text |

This is a reasonable real-world substitute — not every detection's source
data lends itself to clean aggregation, and a "recent alerts" feed is a
normal SOC dashboard pattern for a signal that's inherently closer to a
single raw event than a grouped metric.

## What it currently shows

As of this writing (5-week range), the dashboard shows:

- **Portainer Brute Force: 2 alerts**, both for the same source/destination
  pair (`10.0.50.10` → `10.0.40.10`). This reflects two separate live test
  runs of the brute-force simulation against Portainer during detection
  validation, not two distinct attacker campaigns.
- **DVWA Command Injection: 1 alert**, a real `POST /vulnerabilities/exec/`
  request from `192.168.0.234` at `Nov 2, 2026 @ 18:31:31`. This is a
  genuine detection of the actual command-injection attack traffic from
  this scenario, not a synthetic test fire.

Both detections are confirmed working against real attack traffic, not
just rule-logic test cases — unlike the Kerberoasting dashboard in this
repo, there is no known gap being carried here. The one caveat worth
noting: rule `100151` detects that the vulnerable endpoint was *submitted
to* — it does not inspect the POST body, so it cannot distinguish a benign
form submission from a malicious injection payload by content alone. In
this environment that distinction doesn't matter, since nothing
legitimate should ever POST to this endpoint in the first place — but it's
a real limitation of the detection's design, not an oversight to hide.
