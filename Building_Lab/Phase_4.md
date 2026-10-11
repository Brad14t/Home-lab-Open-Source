# My Process

<img width="385" height="113" alt="image" src="https://github.com/user-attachments/assets/e76abeac-6f1e-4edb-9292-9fbaea3ed84e" />

VULNERABLE firewall rules

Why: Right now VULNERABLE only blocks CORPORATE. This VLAN holds deliberately broken boxes, so it should be default-deny. The only exceptions are the docker host’s internet access during the build, and later its Wazuh agent traffic.

First, create a host alias in Firewall > Aliases: VULN_DOCKER = 10.0.40.10.

<img width="244" height="226" alt="image" src="https://github.com/user-attachments/assets/c312b1bf-ca1a-407d-a49d-89f0a5c9d237" />

# New Firewall Rules

## VULNERABLE VLAN Firewall Rules (10.0.40.0/24)

The VULNERABLE VLAN holds deliberately insecure systems (DVWA, bWAPP, Metasploitable 2), so it runs **default-deny**. Only DNS, gateway ping, Wazuh agent traffic, and reverse shells to the ATTACKER VLAN are allowed out. Internet access is granted to the Docker host only while it is being built, and is disabled afterward.

### Aliases

| Alias | Type | Members | Purpose |
|---|---|---|---|
| `RFC1918` | Network(s) | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | All private ranges |
| `VULN_DOCKER` | Host(s) | 10.0.40.10 | Docker host (DVWA, bWAPP, Portainer) |
| `VULN_MGMT_PORTS` | Port(s) | 22, 9443, 8080, 8081 | SSH, Portainer, DVWA, bWAPP |

### VULNERABLE interface rules

Rules are processed top to bottom; first match wins. Anything not matched hits pfSense's implicit deny.

| # | Action | Protocol | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | VULNERABLE subnets | VULNERABLE address | 53 | No | DNS to pfSense |
| 2 | Pass | ICMP (echo request) | VULNERABLE subnets | VULNERABLE address | n/a | No | Ping gateway |
| 3 | Pass | TCP | VULN_DOCKER | 10.0.20.10 | 1514-1515 | No | Wazuh agent |
| 4 | Block | Any | VULNERABLE subnets | This Firewall | any | Yes | Protect pfSense GUI |
| 5 | Pass | Any | VULNERABLE subnets | ATTACKER subnets | any | No | Reverse shells to attacker VLAN |
| 6 | Block | Any | VULNERABLE subnets | RFC1918 | any | Yes | Block other VLANs and home network |
| 7 | Pass | Any | VULN_DOCKER | any | any | No | **Temporary:** internet for build (disabled after image pulls) |


### Design notes

- **Default-deny:** vulnerable hosts can't reach CORPORATE, SECURITY (except the Wazuh agent port), the home network, or the pfSense GUI.
- **Least privilege on internet access:** rule 7 is scoped to the Docker host only. Metasploitable 2 never has internet access.
- **Temporary rule:** rule 7 is disabled once container images are pulled, leaving the VLAN fully isolated.
- **Logging on block rules:** rules 4 and 6 log, so lateral movement attempts from a compromised host show up in the firewall logs.
- **Attack path preserved:** rule 5 lets exploited hosts call back to Kali in the ATTACKER VLAN, which is needed for realistic purple-team scenarios.


















