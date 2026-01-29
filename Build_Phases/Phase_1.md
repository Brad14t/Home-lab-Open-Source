# Phase 1

# Pre-Build Checklist

- [x] Proxmox VE installed and updated
- [x] At least 64GB RAM available
- [x] 500GB+ storage available
- [x] Network connectivity to Proxmox host
- [x] ISO files downloaded (see ISO Download List below)


# ISO Download List

I start by downloading these ISOs and upload to Proxmox before starting:

```bash
# Core Infrastructure
- pfSense CE (latest): https://www.pfsense.org/download/
- Ubuntu Server 24.04 LTS: https://ubuntu.com/download/server

# Windows (Evaluation versions - 180 days)
- Windows Server 2022 Evaluation: https://www.microsoft.com/en-us/evalcenter/
- Windows 10/11 Enterprise Evaluation: https://www.microsoft.com/en-us/evalcenter/

# Security Tools
- Kali Linux: https://www.kali.org/get-kali/
- Security Onion: https://securityonionsolutions.com/software

# Vulnerable Systems
- Metasploitable 2: https://sourceforge.net/projects/metasploitable/
```

# Step 1: Proxmox Network Configuration

## 1.1 Creating Virtual Bridges
































































































