# Lab Restart Checklist

Use this every time you power the lab back on. The host RAM is overcommitted
(about 57 GB allocated across all VMs vs ~31 GiB usable), so **you do not boot
everything** — you boot the set that matches the task. Pick a boot set below.

Always start **VM 100 (pfSense) first** and give it ~60s before anything else,
since all VLAN traffic routes through it.

---

## Boot sets (pick one — don't exceed host RAM)

| Task | VMs to boot | Approx RAM |
|---|---|---|
| **Monitoring / tuning only** | 100, 101 (+130 for live agent alerts) | ~13-14 GB |
| **AD work / AD attacks** | 100, 101, 110, 111, 120, (121) | ~26 GB |
| **Vulnerable / web-attack work** | 100, 101, 130, 131, 140, 141 | ~24 GB |
| **Single-DC attack (Kerberoast/AS-REP)** | 100, 101, 110, 140 | ~16 GB |
| **TheHive case work** | add 103 then 104 (after 103 is ready) | +9 GB |

> **Security Onion** is deferred until the RAM upgrade — it cannot run alongside
> a full AD-attack set on current hardware.

---

## 1. Power on — always in this order

| Order | VMID | Name | Role | Wait before next? |
|---|---|---|---|---|
| 1 | 100 | soc-fw-01 | pfSense firewall | **Yes — ~60s**, everything routes through it |
| 2 | 101 | soc-wazuh-01 | Wazuh (manager/indexer/dashboard) | No |
| 3 | 110 | corp-dc-01 | DC01 (forest root, DNS, PDC) | **Yes — ~60s** if 111 follows, for replication |
| 4 | 111 | corp-dc-02 | DC02 (additional DC, DNS) | No |
| 5 | 120 / 121 | corp-ws-01/02 | Win11 workstations | No |
| 6 | 130 | vuln-docker-01 | Docker: DVWA, bWAPP, Portainer | No |
| 7 | 131 | vuln-msf2-01 | Metasploitable 2 | No |
| 8 | 140 | atk-kali-01 | Kali + Caldera | No |
| 9 | 141 | atk-nessus-01 | Nessus Essentials | No |
| — | 103 | soc-thehive-data-01 | Cassandra + Elasticsearch | **Yes — wait for 9042/9200** before 104 |
| — | 104 | soc-thehive-app-01 | TheHive + Cortex (manual start, see §3) | No |

Boot only the VMs for your chosen set. Shut down in **reverse order** (attack/target
VMs first, workstations before DCs, app before data).

---

## 2. Management PC static routes (should persist — added with `route -p`)

These were added permanently, so they survive reboot. Only re-add if `route print`
shows them missing (a tell: `tracert` to a lab IP shows hop 1 = 192.168.0.1 instead
of 192.168.0.254).

```
route -p add 10.0.20.0 mask 255.255.255.0 192.168.0.254
```
```
route -p add 10.0.30.0 mask 255.255.255.0 192.168.0.254
```
```
route -p add 10.0.40.0 mask 255.255.255.0 192.168.0.254
```
```
route -p add 10.0.50.0 mask 255.255.255.0 192.168.0.254
```

---

## 3. Verify services after boot

Most services auto-start. The exceptions are **TheHive and Cortex**, which are
disabled at boot by choice and must be started manually, in order.

**VM 101 (Wazuh)** — confirm the core three, then the syslog receiver:
```
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager
```
```
sudo systemctl status rsyslog --no-pager
```
Confirm the ports (agents, API, indexer, dashboard, pfSense syslog receiver):
```
sudo ss -tlnp | grep -E ':(1514|1515|55000|9200|443)\b'
sudo ss -ulnp | grep ':514'
```

**VM 130 (docker host)** — containers auto-restart (`unless-stopped`):
```
sudo docker ps
```

**VMs 140/141 (Kali/Nessus)** — SSH, Caldera, and Nessus auto-start:
```
sudo systemctl status ssh caldera --no-pager   # on Kali
sudo systemctl status nessusd --no-pager        # on Nessus
```

**TheHive case work only — start 103 first, confirm, then start 104:**
```
# On VM 103 (data):
sudo ss -tlnp | grep -E ':(9042|9200)\b'
```
```
# On VM 104 (app) — ONLY after 103's ports are up:
sudo systemctl start cortex
sudo systemctl start thehive
sudo journalctl -u thehive -f --no-pager   # watch until it binds, Ctrl+C when up
```
> If TheHive/Cortex were started before 103 was ready, they fail to connect —
> restart them once 103 is confirmed up.

---

## 4. Confirm web UIs load (from the management PC)

| Service | URL | Notes |
|---|---|---|
| pfSense | https://192.168.0.254 | |
| Wazuh | https://10.0.20.10 | self-signed cert warning expected |
| Splunk | http://10.0.20.11:8000 | **http, not https** |
| TheHive | http://10.0.20.13:9000 | **read-only until Community license is activated** (Platinum trial expired) |
| Cortex | http://10.0.20.13:9001 | |
| Portainer | https://10.0.40.10:9443 | self-signed |
| DVWA | http://10.0.40.10:8080 | |
| bWAPP | http://10.0.40.10:8081 | |
| Caldera | http://10.0.50.10:8888 | log in as `red` |
| Nessus | https://10.0.50.11:8834 | self-signed |

---

## 5. Temporary firewall rules — check before and after attack work

- The **TEMP ATTACKER→CORPORATE rule** (Kali → DC01, ports 88/389) for
  Kerberoast/AS-REP is **normally DISABLED**. Enable it only for the attack run,
  then disable it immediately after.
- The **VULNERABLE temp internet rule** (rule 7, VULN_DOCKER → any) is **normally
  DISABLED**. Enable only to pull updates/images, then disable again.
- After any attack session: confirm both are back to disabled.

---

## 6. Open items / known gotchas when troubleshooting

- **Management PC IP (192.168.0.234) is NOT reserved** — Cox gateway doesn't
  support DHCP reservations. If lab UIs become unreachable, check the PC's IP
  first (`ipconfig`); if it changed, the WAN rules scoped to .234 won't match.
- WAN rule sources are left at `192.168.0.0/24` (not tightened to .234) for the
  same reason — availability over a marginal source restriction.
- Check Proxmox per-VM **Firewall checkbox is OFF** on the NIC (should be off on
  all VMs; a silent-drop cause).
- Service shows "active" but isn't reachable → check the real log file and the
  file ownership for the service's dedicated user (recurring across
  Splunk/TheHive/Cortex). `systemctl status` alone can report stale "active".
- Traffic times out → pfSense → Diagnostics → States, filter by destination IP.
  No state = the rule isn't matching or traffic isn't arriving.
