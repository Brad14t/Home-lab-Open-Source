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

