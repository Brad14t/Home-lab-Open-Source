# Architecture

Full network design, VLAN rationale, and tool stack for the home SOC lab.

---

## Topology

```mermaid
flowchart TB
    NET["☁ Internet / Cox<br/>192.168.0.1"]
    MGMT["🖥 Management PC<br/>192.168.0.234"]

    NET --- FW
    MGMT -.WAN rules.-> FW

    FW["🛡 pfSense — soc-fw-01<br/>WAN 192.168.0.254<br/>5-VLAN segmentation"]

    subgraph SEC["SECURITY VLAN · 10.0.20.0/24"]
        WAZUH["Wazuh SIEM<br/>.10"]
        SPLUNK["Splunk<br/>.11"]
        HIVE["TheHive + Cortex<br/>.12 / .13"]
    end

    subgraph CORP["CORPORATE VLAN · 10.0.30.0/24"]
        DC1["DC01<br/>.10"]
        DC2["DC02<br/>.11"]
        WS["WS01 / WS02<br/>DHCP"]
    end

    subgraph VULN["VULNERABLE VLAN · 10.0.40.0/24"]
        DOCKER["Docker host<br/>DVWA · bWAPP · Portainer<br/>.10"]
        MSF["Metasploitable 2<br/>.20"]
    end

    subgraph ATK["ATTACKER VLAN · 10.0.50.0/24"]
        KALI["Kali + Caldera<br/>.10"]
        NESSUS["Nessus<br/>.11"]
    end

    FW --> SEC
    FW --> CORP
    FW --> VULN
    FW --> ATK

    KALI -.attacks.-> VULN
    KALI -.scoped attacks.-> CORP
    CORP -.agents/logs.-> WAZUH
    VULN -.agent/IDS.-> WAZUH
    WAZUH -.cases.-> HIVE

    classDef sec fill:#0f2417,stroke:#3fb950,color:#56d364
    classDef corp fill:#241e10,stroke:#d29922,color:#e3b341
    classDef vuln fill:#2d1113,stroke:#f85149,color:#ff7b72
    classDef atk fill:#2d1113,stroke:#f85149,color:#ff7b72
    classDef fw fill:#1c2128,stroke:#58a6ff,color:#79c0ff

    class WAZUH,SPLUNK,HIVE sec
    class DC1,DC2,WS corp
    class DOCKER,MSF vuln
    class KALI,NESSUS atk
    class FW fw
```

---

## Host

| Item | Value |
|---|---|
| Hypervisor | Proxmox VE |
| Hardware | HP Elite Mini 600 G9 — 14 cores / 20 threads |
| RAM | 32 GB (upgrade to 64 GB planned) |
| Primary VM storage | 4 TB SATA SSD (qcow2, snapshots enabled) |

---

## Network Segmentation

Five VLANs behind a pfSense firewall. Each VLAN has its own ruleset; the default posture is deny, with explicit passes per service.

| VLAN | Subnet | Purpose | Posture |
|---|---|---|---|
| SECURITY | 10.0.20.0/24 | SOC tooling (Wazuh, Splunk, TheHive, Cortex) | Default pass (trusted) |
| CORPORATE | 10.0.30.0/24 | Active Directory, workstations | DNS/agents/internet only |
| VULNERABLE | 10.0.40.0/24 | DVWA, bWAPP, Metasploitable | Default-deny, no internet |
| ATTACKER | 10.0.50.0/24 | Kali, Nessus | VULNERABLE + internet only |
| Management | 192.168.0.0/24 | Home network / admin PC | pfSense WAN |

Full rulesets: [../firewall-rules/](../firewall-rules/)

---

## Why It's Built This Way

- **Segmentation first** — every VLAN is isolated by default, so an attacker in one segment can't trivially reach another. This is what the detections are tested against.
- **Separate attack and target networks** — mirrors a real engagement where the red team operates from dedicated infrastructure.
- **Host + network telemetry** — endpoint agents (Wazuh/Sysmon) and a network IDS (Suricata) are deployed together, because each sees what the other misses.

---

## Tool Stack

| Function | Tool |
|---|---|
| Hypervisor | Proxmox VE |
| Firewall / router | pfSense |
| Network IDS | Suricata (ET Open ruleset) |
| SIEM | Wazuh (manager + indexer + dashboard) |
| Log platform | Splunk |
| Case management | TheHive + Cortex |
| Endpoint telemetry | Sysmon (sysmon-modular), Wazuh agents |
| Directory services | Windows Server 2025 AD |
| Attack platform | Kali Linux, MITRE Caldera |
| Vulnerability scanner | Nessus Essentials |
| Vulnerable targets | DVWA, bWAPP, Metasploitable 2 |
