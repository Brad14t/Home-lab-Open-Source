# WAN firewall rules (management PC into the lab)

The home network sits on pfSense's WAN, which denies inbound by default, so each
management path needs an explicit pass rule.

| Action | Protocol | Source | Destination | Port | Purpose |
|---|---|---|---|---|---|
| Pass | TCP | 192.168.0.234 | This Firewall | 443 | pfSense web UI |
| Pass | TCP | 192.168.0.0/24 | 10.0.20.0/24 | 443 | Wazuh dashboard |
| Pass | TCP | 192.168.0.0/24 | 10.0.20.0/24 | 8000 | Splunk |
| Pass | TCP | 192.168.0.0/24 | 10.0.20.0/24 | 9000 | TheHive |
| Pass | TCP | 192.168.0.0/24 | 10.0.20.0/24 | 9001 | Cortex |
| Pass | TCP | 192.168.0.0/24 | 10.0.20.0/24 | 22 | SSH to SECURITY VLAN |
| Pass | TCP | 192.168.0.234 | 10.0.30.10 | 22 | SSH to DC01 |
| Pass | TCP | 192.168.0.234 | VULN_DOCKER | VULN_MGMT_PORTS | SSH, Portainer, DVWA, bWAPP |
| Pass | TCP | 192.168.0.234 | ATTACKER_HOSTS | ATK_MGMT_PORTS | SSH, Nessus, Caldera |

**Not exposed by design:**
- No RDP into CORPORATE. Workstations are managed via the Proxmox console only.
- Metasploitable 2 is console-only (no WAN rule).

## Port aliases

| Alias | Ports | Services |
|---|---|---|
| `VULN_MGMT_PORTS` | 22, 9443, 8080, 8081 | SSH, Portainer, DVWA, bWAPP |
| `ATK_MGMT_PORTS` | 22, 8834, 8888 | SSH, Nessus, Caldera |

## Known limitation

The management PC's IP (`192.168.0.234`) is **not reserved** — the ISP gateway
doesn't support DHCP reservations. WAN sources are therefore left at
`192.168.0.0/24` rather than tightened to the single host: a deliberate choice of
availability over a marginal source restriction. If the PC's IP changes, the
host-scoped rules (DC01, docker, attacker hosts) stop matching until it's corrected.
