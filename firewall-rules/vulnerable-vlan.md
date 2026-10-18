# VULNERABLE VLAN firewall rules

Subnet: `10.0.40.0/24` · Interface: OPT2 (vtnet3)

Posture: default-deny. Only DNS/ping to pfSense, the Wazuh agent (docker host only),
and replies to the ATTACKER VLAN are allowed. No internet.

| # | Action | Protocol | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | VULNERABLE subnets | VULNERABLE address | 53 | No | DNS to pfSense |
| 2 | Pass | ICMP (echo request) | VULNERABLE subnets | VULNERABLE address | n/a | No | Ping gateway |
| 3 | Pass | TCP | VULN_DOCKER | 10.0.20.10 | 1514-1515 | No | Wazuh agent |
| 4 | Block | Any | VULNERABLE subnets | This Firewall | any | Yes | Protect pfSense GUI |
| 5 | Pass | Any | VULNERABLE subnets | ATTACKER subnets | any | No | Reverse shells to attacker VLAN |
| 6 | Block | Any | VULNERABLE subnets | RFC1918 | any | Yes | Block other VLANs and home network |
| 7 | Pass (**DISABLED**) | Any | VULN_DOCKER | any | any | No | TEMP internet for build (normally disabled) |

**Rule 7** is normally disabled. Re-enable only to pull updates or new container
images, then disable again immediately.

**Verified:** Metasploitable can ping the gateway and the docker host; it cannot
reach the internet (8.8.8.8) or the CORPORATE VLAN.

> `VULN_DOCKER` alias = `10.0.40.10` (the docker host running DVWA, bWAPP, Portainer).
