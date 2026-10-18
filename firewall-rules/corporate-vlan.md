# CORPORATE VLAN firewall rules

Subnet: `10.0.30.0/24` · Interface: OPT1 (vtnet2)

Rules are processed top to bottom; first match wins. Posture: DNS, Wazuh agents,
and internet only — all other private ranges blocked.

| # | Action | Protocol | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | CORPORATE subnets | CORPORATE address | 53 | No | DNS to pfSense |
| 2 | Pass | ICMP (echo request) | CORPORATE subnets | CORPORATE address | n/a | No | Ping gateway |
| 3 | Pass | TCP | CORPORATE subnets | 10.0.20.10 | 1514-1515 | No | Wazuh agents |
| 4 | Block | Any | CORPORATE subnets | This Firewall | any | Yes | Protect pfSense GUI |
| 5 | Block | Any | CORPORATE subnets | RFC1918 | any | Yes | Block other VLANs and home net |
| 6 | Pass | Any | CORPORATE subnets | any | any | No | Internet |

**Verified from DC01:** DNS, ping, and Wazuh 1514 pass; Wazuh 443 and the pfSense
GUI are blocked; internet passes.

> `RFC1918` alias = `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` (all private ranges).
