# Summarized Build Log

Condensed summary of how the lab was built, phase by phase. This is the "what and why," not a step-by-step tutorial.

---

## Phase 1 — Network Infrastructure

pfSense firewall with five VLANs, segmentation rulesets, and hardware offloading disabled for Suricata. Management PC routed into the lab via static routes.

**Key outcomes:** isolated VLANs, default-deny posture, verified traffic boundaries.

---

## Phase 2 — Security Monitoring

Wazuh all-in-one (manager, indexer, dashboard), Splunk, and case management. Case management was first built as TheHive + Cortex (split across a data VM and an app VM, with a custom Wazuh → TheHive integration for alert-to-case forwarding). It was later replaced by **DFIR-IRIS** on the same two VMs (103/104), which is the case platform used in every scenario.

**Key outcomes:** working SIEM, case management pipeline.

---

## Phase 3 — Active Directory

Two domain controllers (`lab.local`), OU structure, users, groups, Group Policy (password/audit/PowerShell/firewall), and Windows 11 workstations joined. Sysmon (sysmon-modular) and Wazuh agents on all Windows hosts.

**Deliberately vulnerable accounts:** `svc-sql` (Kerberoastable), `svc-backup` (AS-REP roastable) — targets for Phase 6.

**Key outcomes:** realistic AD environment with endpoint telemetry.

---

## Phase 4 — Vulnerable Systems

Docker host running DVWA, bWAPP, and Portainer; Metasploitable 2 imported and isolated. A dedicated Suricata instance on the VULNERABLE VLAN with a scoped Home Net.

**Key outcomes:** attackable targets, network IDS coverage.

---

## Phase 5 — Attack Infrastructure

Kali Linux with MITRE Caldera (systemd service) and Nessus Essentials. A baseline vulnerability scan of the VULNERABLE VLAN, observed in both Suricata (network) and Wazuh (host logs).

**Key outcomes:** working attack platform, proof that both telemetry sources see activity.

---

## Phase 6 — Detection Engineering / Purple Team

Alert tuning, Suricata → Wazuh forwarding, and scoped attack scenarios with documented detections.

**Key outcomes:** see [../scenarios/](../scenarios/).

---

## Troubleshooting

Problems hit during the build, with root causes and fixes: [troubleshooting.md](troubleshooting.md)
