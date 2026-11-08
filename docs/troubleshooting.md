# Troubleshooting Log

Every non-trivial problem hit during the build, with the symptom, the root cause, and the fix. These are the moments where the lab taught something — the most useful part of the project for anyone building the same thing.

| # | Symptom | Root cause | Fix |
|---|---|---|---|
| 1 | New VM couldn't ping its gateway | Proxmox per-VM firewall on by default, silently dropping traffic | Uncheck Firewall on the VM's NIC; pfSense handles segmentation |
| 2 | Ubuntu LVM gave `/` only half the disk | Installer default | Resize `ubuntu-lv` to the full volume group during install |
| 3 | Dashboard unreachable from the management PC | Home network is pfSense's WAN, which denies inbound by default | Add a WAN pass rule per service port |
| 4 | WAN rule showed 0 states, never matched | Source entered as /32 instead of /24 | Correct the CIDR mask; diagnose with packet capture + Diagnostics > States |
| 5 | Traffic to the SOC VLAN went out to the home router | Windows static route missing (doesn't survive reboot/sleep) | Add the route with `route -p`; `tracert` hop 1 = home router is the tell |
| 6 | Splunk web UI timed out | Used https instead of http | Splunk's UI is http on port 8000 |
| 7 | Splunk service wouldn't start (port in use) | First run as root; service runs as `splunk`; stale root PIDs held the port | Kill leftover PIDs, `chown -R splunk:splunk /opt/splunk`, start via systemd |
| 8 | `systemctl status` showed "active (exited)" | LSB init-script wrapper reports a stale state | Verify with `ss -tlnp` and the real process/log, not systemd alone |
| 9 | Cassandra "Unable to gossip with any peers" | `listen_address` changed but `seeds` still 127.0.0.1 | Set both to the VM's IP |
| 10 | Nano showed an empty Elasticsearch config | Edited `.yaml`; real file is `.yml`; nano silently made a new file | Edit the correct path; confirm with `dpkg -L` |
| 11 | TheHive/Cortex apt repos unavailable | Vendor deprecated the apt/yum repos | Download `.deb` files via each product's `manifest.json` |
| 12 | Cortex crashed with exit 255 | `/opt/cortex` root-owned; the `cortex` user couldn't write its PID file | `chown -R cortex:cortex /opt/cortex`; found by running in foreground |
| 13 | Cortex analyzer list empty | Catalog not loaded and Docker not installed | Install Docker, add `cortex` to the `docker` group, refresh analyzers |
| 14 | TheHive page wouldn't load though everything looked fine | Data VM wasn't running | Boot the data VM before the app VM; restart TheHive/Cortex if they started early |
| 15 | Multi-line pastes got flattened in the console | Frozen console session + paste flattening | Reconnect; prefer single-line commands and SSH over the console |
| 16 | Windows VM fell through to PXE instead of the installer | OVMF boot-from-CD prompt is only a few seconds; boot-order changes stay pending until a full stop | `qm stop`, then start and send Enter keys in a loop; ISO first, net0 unchecked |
| 17 | `dcdiag` failed right after DC promotion | First-boot timing: Netlogon/RID allocator ran before DNS and AD were fully up | Confirm DNS points to self, restart Netlogon, re-test targeted dcdiag |
| 18 | SSH to DC01 timed out though sshd was listening | OpenSSH firewall rule scoped to Private profile; a DC's NIC uses Domain | `Set-NetFirewallRule -Name OpenSSH-Server-In-TCP -Profile Any` |
| 19 | Domain join + rename in one step failed ("directory service busy") | Rename sent to a DC that hadn't replicated the new computer object | Rename first, reboot, then join |
| 20 | sysmon-modular config download returned 404 | Project moved prebuilt configs to GitHub Release assets | Use the `releases/latest/download/` URL |
| 21 | Wazuh agent install "succeeded" but the service didn't exist | `msiexec /q` hides failures and returns immediately | Confirm the MSI, copy locally, run with `-Wait -PassThru` and `/l*v` logging, check exit code |
| 22 | `adapter "Ethernet" not found` on DC02 | NIC had a different name ("Ethernet 2") | Query the live adapter name dynamically and use `-InterfaceAlias` |
| 23 | Suricata warned offloading must be disabled | Checksum/TSO/LRO offloading still enabled on pfSense | Disable all three in System > Advanced > Networking, reboot |
| 24 | Suricata ran with only built-in rules | ET Open never enabled in Global Settings | Enable ETOpen, set a daily update, select categories per interface, restart |
| 25 | Archive searches returned nothing for events that clearly happened (sshd logons, agent `vuln-docker-01`) | `logall_json` had been off since 2026-10-31, so `archives.json` was a stale file and every search ran against 8-day-old data | Set `<logall_json>yes</logall_json>` in `ossec.conf`, validate, restart. Check the last timestamp before trusting an archive search: `tail -1 archives.json \| jq -r .timestamp` |
| 26 | `zgrep` on `/var/ossec/logs/archives/2026/Nov/*.json.gz` said "No such file" | The non-root shell expands the wildcard before `sudo` runs, and the archives directory isn't readable by the user | Run the whole command as root (`sudo bash -c '…'`), or list with `sudo ls` first |
| 27 | Discover query `rule.id:60106 and …targetUserName:jcarter` returned 0, though `jcarter` logons existed | Wazuh routes 4624 to different child rules by logon type and auth package (67022 local, 92651/92652 remote and NTLM), and an alert carries only the final rule ID | Query by `data.win.system.eventID:4624` to discover the real rule IDs; list all parents in custom rules (100160) |
| 28 | Agent showed "Active" but appeared to send no events | Misdiagnosis caused by #25 (stale archives) and the wrong agent name in the query (`vulndocker01` vs. `vuln-docker-01`). `wazuh-agentd.state` showed tens of thousands of messages sent and a current ACK | Use `agent_control -l` for exact agent names; check `/var/ossec/var/run/wazuh-agentd.state` before blaming the agent |
| 29 | auditd "active" but `auditctl -l` showed **No rules**; Scenario 03's auditd watches were gone | Watches were added at runtime with `auditctl` and never written to `/etc/audit/rules.d/`, so a reboot wiped them. `audit.log` also isn't in the agent's localfile list | Pending (sensitive-file block): persist rules in `/etc/audit/rules.d/*.rules`, add an `audit.log` localfile, verify after a reboot |
| 30 | Threat-indicator tile showing **1** stayed green | Dashboard metric color range `0 → 1` includes the value 1 | Use `0 → 0.5` green and `0.5 → 100000` red |
| 31 | Dashboard saved with a pinned `agent.name: is one of …` filter | Selecting every host in the Controls panel creates a real filter; a later agent would be silently excluded | Clear the control and remove the filter pill before saving. Empty means all hosts |
| 32 | Risk found in audit: vuln-docker-01's agent reads the DVWA log by its full container-ID path | The path belongs to one container instance; recreating the container changes the ID and breaks Scenario 03's rules 100150/100151 silently | Re-point the localfile after any container recreation, or use a wildcard path |
| 33 | Risk found while building 100161: custom child rules can stop Wazuh's built-in brute-force correlation from counting | A custom child rule takes over the event from its built-in parent, so the built-in frequency rule no longer sees those failures | Pair every custom failure rule with its own frequency rule (100162, 100167) |
| 25 | Test attacks produced no Suricata alerts | Default Home Net trusted all local networks, so external→home rules never matched | Custom Pass List with only the target subnet as Home Net |
| 26 | New Kali VM couldn't ping its gateway | A firewall rule's source was copied from another VLAN's ruleset | Correct the source; `ip neigh` showing a resolved MAC points to the firewall, not L2 |
| 27 | After Kali install, the interface was down with no address | Text-mode installer's network step didn't complete | Add a static stanza to `/etc/network/interfaces`, set resolv.conf, restart networking |
| 28 | `apt install` couldn't find packages on Kali | `sources.list` only pointed at the install media | Comment out the cdrom line, add the kali-rolling mirror, update |
| 29 | SSH to Kali refused | OpenSSH server wasn't installed (offline install) | Install and enable `openssh-server`; "refused" proved the firewall path was fine |
| 30 | ~1,650 alerts/day flooding the SIEM | Package-install events (dpkg) and a one-off upgrade spike rated at level 7 | Tune by rule, not by raising the global threshold — drop dpkg/SCA/FIM noise to low levels, keep real detections |
| 31 | Rootcheck flagged a "trojaned" system binary | Generic signature false-positives on a stock binary | Verify with `dpkg --verify`, then suppress that one file at level 0 |
| 32 | Suricata alerts reached the SIEM as level-2 "Unknown problem" | pfSense syslog has no hostname field, breaking the stock decoder | Custom parent/child decoder keyed on the message structure |
| 33 | Syslog listener showed no events despite packets on the wire | The SIEM's built-in syslog listener wasn't ingesting them | Switch to rsyslog as the receiver, writing to a file the SIEM reads as a localfile |
| 34 | rsyslog couldn't create its log file | `/var/log` not group-writable for the `syslog` user | Dedicated log directory owned by `syslog` |
| 35 | pfSense stopped sending syslog mid-session | Sender stalled after a syslog daemon restart | Re-save Status > System Logs > Settings |

---

## Recurring Lessons

- **"active" is not "working."** Always confirm a service with `ss -tlnp`, the real log, and a functional test — not `systemctl status` alone.
- **Tune detections by rule, never by raising the global threshold.** A higher threshold hides the detections you want alongside the noise.
- **File ownership breaks services quietly.** Several tools failed because a dedicated service user couldn't write where it needed to.
- **Confirm a data source is live before concluding something is absent.** A stale archive or a wrong agent name looks exactly like "no events" (#25, #28).
- **The network path is usually innocent.** A resolved ARP entry or a "connection refused" (vs. a timeout) proves L2/routing is fine and points at the firewall or the service.
