# ATTACKER VLAN firewall rules

Subnet: `10.0.50.0/24` · Interface: OPT3 (vtnet4)

Posture: can reach the VULNERABLE VLAN and the internet only. CORPORATE, SECURITY,
and the home network are blocked.

| # | Action | Protocol | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | ATTACKER subnets | ATTACKER address | 53 | No | DNS to pfSense |
| 2 | Pass | ICMP (echo request) | ATTACKER subnets | ATTACKER address | n/a | No | Ping gateway |
| 3 | Block | Any | ATTACKER subnets | This Firewall | any | Yes | Protect pfSense GUI |
| 4 | Pass | Any | ATTACKER subnets | VULNERABLE subnets | any | No | Attack targets |
| 5 | Block | Any | ATTACKER subnets | RFC1918 | any | Yes | Block CORPORATE, SECURITY, home network |
| 6 | Pass | Any | ATTACKER subnets | any | any | No | Internet (updates, Nessus plugins) |

**Verified from Kali and Nessus:** VULNERABLE and the internet pass; CORPORATE and
SECURITY are blocked.

## Temporary purple-team rule (Phase 6)

For AD attack scenarios, a narrow, temporary pass is added **above rule 5**, scoped
to one host and the ports the technique needs. It is **disabled except during an
active run**, and disabled again immediately after.

| Action | Protocol | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|
| Pass | TCP | 10.0.50.10 (Kali) | 10.0.30.10 (DC01) | 88, 389 | Yes | TEMP — Kerberoast/AS-REP — DISABLE AFTER |

> `ATTACKER_HOSTS` alias = `10.0.50.10` (Kali), `10.0.50.11` (Nessus).
