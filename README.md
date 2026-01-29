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

# 🛠️ Technology Stack

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


# 📋 Project Phases

**Phase 1: Foundation & Network Infrastructure**
- [ ] Proxmox VE installation and configuration
- [ ] Network planning and VLAN design
- [ ] pfSense installation and VLAN configuration
- [ ] DNS and DHCP setup
- [ ] Network documentation

**Phase 2: Security Monitoring Infrastructure**
- [ ] Wazuh server deployment
- [ ] Splunk deployment (500MB/day free tier)
- [ ] TheHive + Cortex integration
- [ ] Security Onion deployment
- [ ] Log collection architecture

**Phase 3: Active Directory Environment**
- [ ] Windows Server deployment (DC1 and DC2)
- [ ] Active Directory forest configuration
- [ ] Group Policy Objects (GPOs)
- [ ] User and computer management
- [ ] Windows workstation deployment and domain join

**Phase 4: Endpoint Monitoring**
- [ ] Wazuh agent deployment to all endpoints
- [ ] Sysmon installation and configuration
- [ ] Log forwarding to SIEM platforms
- [ ] Baseline monitoring dashboards

**Phase 5: Vulnerable Application Environment**
- [ ] DVWA deployment
- [ ] bWAPP deployment
- [ ] Metasploitable setup
- [ ] Network isolation and access controls

**Phase 6: Attack Infrastructure**
- [ ] Kali Linux setup
- [ ] Nessus Essentials installation
- [ ] Caldera deployment and configuration
- [ ] Attack simulation planning

**Phase 7: Detection & Response**
- [ ] MITRE ATT&CK mapping
- [ ] Detection rule development
- [ ] Incident response playbooks
- [ ] Purple team exercises

**Phase 8: Automation & Documentation**
- [ ] Automated alert workflows
- [ ] Response automation scripts
- [ ] Complete documentation
- [ ] Portfolio presentation


# 🎓 Skills Demonstrated

- **SIEM Management**: Multi-platform log aggregation, correlation, and analysis
- **Threat Detection**: Creating and tuning detection rules based on MITRE ATT&CK
- **Incident Response**: Using TheHive for case management and investigation
- **Network Security**: Firewall configuration, IDS/IPS implementation
- **Vulnerability Management**: Scanning, assessment, and remediation
- **Active Directory**: Enterprise domain management and security
- **Endpoint Security**: EDR deployment and monitoring
- **Security Automation**: Integration of security tools and automated responses
- **Purple Teaming**: Coordinated attack simulation and defense validation
- **Technical Documentation**: Professional lab documentation for stakeholders

# 🔗 Key Resources

- **MITRE ATT&CK Framework**: https://attack.mitre.org/
- **Wazuh Documentation**: https://documentation.wazuh.com/
- **Splunk Free**: https://www.splunk.com/en_us/download/splunk-enterprise.html
- **TheHive Project**: https://thehive-project.org/
- **Security Onion**: https://securityonionsolutions.com/
- **Caldera**: https://github.com/mitre/caldera


# 📝 Important Notes

- **Cost**: $0 - All tools use free/open-source versions or evaluation licenses
- **Legal**: This lab is for educational purposes in an isolated environment
- **Scalability**: Start with core components, add complexity gradually
- **Documentation**: Each phase includes detailed screenshots and configurations


# 🤝 Connect

This project is part of my cybersecurity portfolio demonstrating hands-on blue team capabilities. I'm actively seeking opportunities in SOC, Threat Detection, Incident Response, and Security Engineering roles.





























































