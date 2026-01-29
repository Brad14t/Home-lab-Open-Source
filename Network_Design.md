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













