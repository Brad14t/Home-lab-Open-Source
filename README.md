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

<img width="385" height="173" alt="image" src="https://github.com/user-attachments/assets/687df5b8-ed67-442b-9be8-f67c8fa64855" />

# Infrastructure Components 

**Security Monitoring & SIEM (VLAN 20)**

- **Wazuh** - Open-source SIEM and XDR platform
- **Splunk** - Log aggregation and analysis (Free license: 500MB/day)
- **TheHive** - Incident response platform
- **Cortex** - Observable analysis and active response engine
- **Security Onion** - Network security monitoring
- **Elastic Stack (ELK)** - Log management integrated with Wazuh

**Endpoint Systems (VLAN 30)**

- **Active Directory Server** - Windows Server 2019/2022 (Evaluation)
- **Domain Controllers** - Primary and backup DC
- **Windows 10/11 Workstations** - Corporate endpoints (Evaluation)
- **Ubuntu Desktop** - Linux workstation endpoint

**Vulnerable Applications (VLAN 40)**

- **DVWA** - Damn Vulnerable Web Application
- **bWAPP** - Buggy Web Application
- **VulnHub VMs** - Various vulnerable systems
- **Metasploitable** - Intentionally vulnerable Linux VM

**Attack Simulation (VLAN 50)**

- **Kali Linux** - Penetration testing platform
- **Nessus Essentials** - Vulnerability scanner (Free version)
- **Caldera** - Adversary emulation platform (MITRE ATT&CK)

**Supporting Infrastructure**

- **pfSense** - Firewall/Router with Suricata IDS
- **Docker Host** - Container services (Portainer for management)
- **Ubuntu Server** - General-purpose server hosting

## 🛠️ Technology Stack

| Category | Tools |
|----------|-------|
| **Hypervisor** | Proxmox VE |
| **SIEM/XDR** | Wazuh, Splunk, ELK Stack |
| **Incident Response** | TheHive, Cortex |
| **NSM** | Security Onion, Suricata |
| **Endpoint Detection** | Wazuh Agents, Sysmon |
| **Vulnerability Scanning** | Nessus Essentials, OpenVAS |
| **Penetration Testing** | Kali Linux, Metasploit |
| **Adversary Emulation** | Caldera (MITRE ATT&CK) |
| **Network Security** | pfSense, Suricata IDS/IPS |
| **Containerization** | Docker, Portainer |
| **Directory Services** | Active Directory |
| **Scripting/Automation** | Python, Bash, PowerShell |


## 📋 Project Phases

**Phase 1: Foundation & Network Infrastructure**
- [x] Proxmox VE installation and configuration
- [x] Network planning and VLAN design
- [ ] pfSense installation and VLAN configuration
- [ ] DNS and DHCP setup
- [ ] Network documentation













































































