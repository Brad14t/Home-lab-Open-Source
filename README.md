# Home-lab-NEW
This project documents the complete build and operation of an enterprise-grade cybersecurity lab environment designed for blue team operations, threat detection, incident response, and security monitoring.

# The Ultimate Blue Team Cybersecurity Lab

![Lab Status](https://img.shields.io/badge/status-in--progress-yellow)

![Platform](https://img.shields.io/badge/platform-Proxmox-orange)

![Cost](https://img.shields.io/badge/cost-free-green)


# 🎯 Project Overview

This project documents the complete build and operation of an enterprise-grade cybersecurity lab environment designed for blue team operations, threat detection, incident response, and security monitoring. The lab simulates a real-world corporate network with integrated SIEM solutions, vulnerable applications, and automated adversary emulation.

**Purpose**: Demonstrate practical blue team cybersecurity skills including threat detection, log analysis, incident response, vulnerability management, and security tool integration.

# Network Topology

Internet
    │
    ├── pfSense Firewall (10.0.0.1)
    │       │
    │       ├── VLAN 10: Management (10.0.10.0/24)
    │       ├── VLAN 20: Security Tools (10.0.20.0/24)
    │       ├── VLAN 30: Corporate Network (10.0.30.0/24)
    │       ├── VLAN 40: Vulnerable Apps (10.0.40.0/24)
    │       └── VLAN 50: Attacker Network (10.0.50.0/24)

