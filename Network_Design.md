# Network Design & Architecture

# Overview

This document details the complete network architecture for the Ultimate Blue Team Cybersecurity Lab. The design emphasizes network segmentation, security monitoring visibility, and realistic enterprise topology.

# Design Principles

1. **Network Segmentation**: VLANs isolate different security zones
2. **Defense in Depth**: Multiple layers of security controls
3. **Visibility**: All traffic flows through monitoring points
4. **Containment**: Vulnerable systems isolated from production
5. **Realistic Simulation**: Mirrors enterprise network architecture

# VLAN Architecture

| VLAN ID | Network Name | Subnet | Gateway | Purpose |
|---------|-------------|---------|---------|---------|
| 1 | Management | 10.0.10.0/24 | 10.0.10.1 | Proxmox management, admin access |
| 10 | Security Tools | 10.0.20.0/24 | 10.0.20.1 | SIEM, monitoring, security platforms |
| 20 | Corporate Network | 10.0.30.0/24 | 10.0.30.1 | AD, workstations, servers |
| 30 | Vulnerable Apps | 10.0.40.0/24 | 10.0.40.1 | DVWA, bWAPP, intentionally vulnerable systems |
| 40 | Attacker Network | 10.0.50.0/24 | 10.0.50.1 | Kali, Nessus, attack simulation tools |
| 99 | WAN/Internet | DHCP | - | External connectivity (optional) |

# Detailed IP Allocation

**VLAN 1 - Management (10.0.10.0/24)**

```
10.0.10.1       - pfSense Management Interface
10.0.10.2       - Proxmox Host
10.0.10.10-20   - Reserved for infrastructure
```

**VLAN 10 - Security Tools (10.0.20.0/24)**

```
10.0.20.1       - Gateway (pfSense)
10.0.20.10      - Wazuh Manager
10.0.20.11      - Wazuh Indexer
10.0.20.12      - Wazuh Dashboard
10.0.20.20      - Splunk Enterprise
10.0.20.30      - TheHive
10.0.20.31      - Cortex
10.0.20.40      - Security Onion
10.0.20.50      - Docker Host (Portainer)
10.0.20.100-200 - DHCP Pool for additional security tools
```

**VLAN 20 - Corporate Network (10.0.30.0/24)**

```
10.0.30.1       - Gateway (pfSense)
10.0.30.10      - DC01 (Primary Domain Controller)
10.0.30.11      - DC02 (Secondary Domain Controller)
10.0.30.20      - File Server
10.0.30.30      - Ubuntu Server (internal services)
10.0.30.100-200 - DHCP Pool for workstations
```

**VLAN 30 - Vulnerable Apps (10.0.40.0/24)**

```
10.0.40.1       - Gateway (pfSense)
10.0.40.10      - DVWA
10.0.40.11      - bWAPP
10.0.40.12      - Metasploitable 2
10.0.40.13      - Metasploitable 3
10.0.40.20-50   - VulnHub VMs
```

**VLAN 40 - Attacker Network (10.0.50.0/24)**

```
10.0.50.1       - Gateway (pfSense)
10.0.50.10      - Kali Linux (Primary)
10.0.50.20      - Nessus Scanner
10.0.50.30      - Caldera Server
10.0.50.40      - Ubuntu Attack Box
```

# Firewall Rules & Network Policies

| Source VLAN | Destination VLAN | Access | Purpose |
|-------------|------------------|--------|---------|
| Management | All | ALLOW | Admin access to all systems |
| Security Tools | Corporate | ALLOW | Log collection, monitoring |
| Security Tools | Vulnerable Apps | ALLOW | Monitor vulnerable systems |
| Security Tools | Attacker | ALLOW | Monitor attack traffic |
| Corporate | Security Tools | ALLOW | Send logs to SIEM |
| Corporate | Internet | ALLOW | Normal business operations |
| Vulnerable Apps | Corporate | DENY | Prevent lateral movement |
| Vulnerable Apps | Security Tools | ALLOW | Send logs only |
| Vulnerable Apps | Internet | DENY | Isolation |
| Attacker | Corporate | CONTROLLED | Purple team exercises only |
| Attacker | Vulnerable Apps | ALLOW | Attack simulation |
| Attacker | Security Tools | DENY | Cannot attack monitoring |

# pfSense Firewall Configuration

## Default Rules by VLAN

**Management VLAN**
```
Priority 1: Allow all outbound
Priority 2: Allow access to all VLANs
Priority 3: Log all traffic
```

**Security Tools VLAN**
```
Priority 1: Allow inbound from all VLANs (for log collection)
Priority 2: Allow outbound to all VLANs (for monitoring)
Priority 3: Allow Internet access (updates, threat intel)
Priority 4: Log all traffic
```

**Corporate VLAN**
```
Priority 1: Allow outbound to Internet
Priority 2: Allow to Security Tools (log shipping)
Priority 3: Allow intra-VLAN communication
Priority 4: DENY to Vulnerable Apps
Priority 5: DENY to Attacker Network
Priority 6: Log all traffic
```

**Vulnerable Apps VLAN**
```
Priority 1: Allow to Security Tools (logging only)
Priority 2: DENY to all other VLANs
Priority 3: DENY Internet access
Priority 4: Log all traffic
```

**Attacker Network VLAN**
```
Priority 1: Allow to Vulnerable Apps
Priority 2: CONTROLLED access to Corporate (requires manual enable)
Priority 3: DENY to Security Tools
Priority 4: Allow Internet (for tool updates)
Priority 5: Log all traffic
```

# Traffic Flow & Monitoring

## Log Flow Architecture

```
Endpoints (All VLANs)
    │
    ├──> Wazuh Agents ──> Wazuh Manager (10.0.20.10)
    │                          │
    │                          ├──> Wazuh Indexer (10.0.20.11)
    │                          └──> TheHive (10.0.20.30)
    │
    ├──> Syslog ──> Splunk (10.0.20.20)
    │
    └──> Windows Event Forwarding ──> Splunk / Wazuh
```

## Network Traffic Monitoring

```
All VLAN Traffic
    │
    ├──> pfSense (Suricata IDS) ──> Logs to Security Onion
    │
    └──> Mirror Port ──> Security Onion (10.0.20.40)
                              │
                              ├──> Zeek (Network Analysis)
                              ├──> Suricata (IDS/IPS)
                              └──> PCAP Storage
```




























































































































