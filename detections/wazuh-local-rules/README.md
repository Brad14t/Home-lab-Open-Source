# Wazuh Local Rules & Decoders

All custom detection content on the Wazuh manager (`soc-wazuh-01`, 10.0.20.10).

| File on the manager | Contents |
|---|---|
| `/var/ossec/etc/rules/local_rules.xml` | Custom rules `100100`–`100177` |
| `/var/ossec/etc/decoders/local_decoder.xml` | Custom pfSense decoders |

---

## Rules summary

| Rule ID | Level | MITRE | Purpose | Status |
|---|---|---|---|---|
| 100100 | 3 | — | Dpkg package activity, tuned down from 7 and kept for audit | ✅ |
| 100101 | 0 | — | Rootcheck false positive: trojan signature on stock `diff` (verified with `dpkg --verify`) | ✅ |
| 100102 | 3 | — | SCA check failure, tuned down from 7 | ✅ |
| 100103 | 3 | — | FIM change in package-managed paths (certs, boot, `ld.so.cache`), tuned down from 7 | ✅ |
| 100110 | 3 | — | pfSense Suricata alert (base) | ✅ |
| 100111 | 5 | — | pfSense Suricata priority 2 | ✅ |
| 100112 | 8 | — | pfSense Suricata priority 1 (HIGH, reaches case management) | ✅ |
| 100120 | 2 | — | pfSense firewall block (base) | ✅ |
| 100121 | 7 | — | Block from ATTACKER VLAN (probing beyond allowed scope) | ✅ |
| 100122 | 7 | — | Block from VULNERABLE VLAN (compromised host attempting egress) | ✅ |
| 100130 | 12 | T1558.003 | Kerberoasting: RC4 service ticket requested | ⚠️ Gap (see below) |
| 100131 | 10 | T1558.003 | Kerberoasting: ticket requested for a monitored `svc-*` account | ⚠️ Gap (see below) |
| 100132 | 12 | T1558.004 | AS-REP roasting: TGT requested without pre-authentication | ✅ |
| 100140 | 3 | — | Allowed connection to Portainer (9443) on vuln-docker-01 (base) | ✅ |
| 100141 | 10 | T1110 | Portainer brute force: 5+ connections to 10.0.40.10:9443 from one source in 60s | ⚠️ Perimeter only |
| 100150 | 3 | — | DVWA: POST to the command-injection endpoint (base) | ✅ |
| 100151 | 12 | T1190 | Possible command-injection exploitation on DVWA | ⚠️ Can't see POST body |
| 100160 | 6 | T1078.002 | Windows logon by any human account | ✅ |
| 100161 | 7 | T1110 | Windows logon failure by any human account | ✅ |
| 100162 | 10 | T1110.001 | Windows brute force: 4+ failures for one account within 2 min | ✅ |
| 100163 | 10 | T1078.002 | Service account (`svc-*`) interactive logon | ✅ |
| 100164 | 5 | T1078.003 | Linux logon (SSH or console) | ✅ |
| 100165 | 7 | T1110 | Linux logon failure | ✅ |
| 100166 | 6 | T1548.003 | Linux sudo to root | ✅ |
| 100167 | 10 | T1110.001 | Linux brute force: 5+ failures from one source IP within 2 min | ✅ |
| 100168 | 12 | T1078.002 | Successful Windows logon from the ATTACKER VLAN (10.0.50.0/24) | ⚠️ Untested (needs temp ATTACKER → DC rule) |
| 100169 | 12 | T1078.003 | Successful Linux logon from the ATTACKER VLAN (10.0.50.0/24) | ✅ |
| 100170 | 12 | T1003.008 | Linux: `/etc/shadow` or `/etc/gshadow` read by an unexpected process (auditd key `sens_shadow`) | ✅ |
| 100171 | 10 | T1098.004, T1053.003, T1548.003 | Linux: change to sudoers, `authorized_keys` or cron files (auditd keys `sens_priv`, `sens_ssh`, `sens_persist`) | ✅ |
| 100172 | 7 | — | Linux: Docker CLI executed on the host (auditd key `sens_docker_cli`) | ✅ |
| 100173 | 12 | T1611 | Linux: `mount` by a non-runtime process, possible container escape (auditd key `sens_mount`) | ✅ |
| 100174 | 12 | T1003.003 | Windows: `vssadmin create shadow`, `esentutl /y` or `ntdsutil` command line (Sysmon event 1) | ✅ |
| 100175 | 12 | T1003.003 | Windows: `ntdsutil` NTDS access (Security 4688) | ✅ |
| 100176 | 12 | T1003.002 | Windows: `reg save` of the SYSTEM or SECURITY hive (Sysmon event 1) | ✅ |
| 100177 | 12 | T1003.001 | Windows: lsass opened with dump-capable access rights (Sysmon event 10) | ⚠️ Unvalidated on live traffic |

Built-in rules the Sensitive File Access dashboard also uses (not modified): `92026` (level 14, `reg save` of the SAM hive) and `92900` (level 12, lsass access with rights `0x1010` or `0x40`).

**Next free ID:** `100178` (reserved for Windows 4663 file-read detection, which needs an audit policy first; the block `100170`–`100179` is sensitive file access and credential dumping).

---

## 100100–100103: Noise tuning

Built in Phase 6 to lower the alert volume of known-benign activity without raising the global alert threshold. Each rule re-levels a built-in rule. 100100, 100102 and 100103 drop to level 3, so they remain low-priority alerts. 100101 drops to level 0, so it isn't alerted at all and is only visible in the archives.

## 100110–100122: pfSense (Suricata + filterlog)

| Rules | Log source | Notes |
|---|---|---|
| 100110–100112 | pfSense Suricata fast-log → rsyslog → Wazuh localfile | Only priority 1 (100112, level 8) reaches case management |
| 100120–100122 | pfSense filterlog CSV → rsyslog → Wazuh localfile | Blocks from ATTACKER (100121) and VULNERABLE (100122) are raised to level 7 |

## 100130–100132: Kerberos credential attacks (Scenarios 01–02)

| Rule | Log source | Notes |
|---|---|---|
| 100130 / 100131 | Windows 4769 (DCs) | **Known gap:** doesn't fire on remote traffic, where Windows reports the client as `::ffff:x.x.x.x`. 100130 also can't fire on this domain because RC4 is disabled (tickets come back AES256). The compensating control is hunting. See [Scenario 01](../../scenarios/01-kerberoasting/) |
| 100132 | Windows 4768 with `preAuthType` 0 (parent `60103`) | Validated on live traffic. See [Scenario 02](../../scenarios/02-asrep-roasting/) |

## 100140–100151: Web attacks (Scenario 03)

| Rule | Log source | Limitation |
|---|---|---|
| 100140 / 100141 | pfSense filterlog | Only sees traffic that crosses pfSense. Brute force from inside the VULNERABLE VLAN or from the host itself is invisible |
| 100150 / 100151 | DVWA Apache access log (container log on vuln-docker-01) | Detects the POST but not the payload: a benign test and a reverse shell produce identical alerts |

See [Scenario 03](../../scenarios/03-web-attacks/command-injection/).

## 100160–100169: Authentication monitoring

These feed the **SOC — Authentication Overview** dashboard. All rules share the group `lab_accounts`, so `rule.groups:lab_accounts` selects the whole block.

### Parents and logic

| Rule | Parent rule(s) | Logic |
|---|---|---|
| 100160 | 60106, 67022, 92651, 92652 | 4624 for a human account. Windows routes 4624 to different built-in children depending on logon type and auth package, so all of them are listed. `if_group authentication_success` was rejected because 67022 is not in that group |
| 100161 | 60122 | 4625 (bad password or unknown user) for a human account |
| 100162 | frequency on 100161 | `same_field` on `targetUserName`, 4 failures in 120s |
| 100163 | 100160 | Username `^svc-` and logon type 2, 10 or 11 |
| 100164 | 5715, 5501 | `Accepted ` (sshd) or `pam_unix(login:session)` (console). PAM sshd and sudo sessions are excluded so one login = one alert |
| 100165 | 5760, 5503 | `Failed `, `Invalid user`, or PAM failures that aren't from sshd, so one SSH failure = one alert |
| 100166 | 5402 | Successful sudo to root. Captures `srcuser` and `command` |
| 100167 | frequency on 100165 | `same_srcip`, 5 failures in 120s |
| 100168 | 100160 | `ipAddress` matches `(^|:)10\.0\.50\.\d{1,3}$`, which handles both `10.0.50.x` and `::ffff:10.0.50.x` |
| 100169 | 100164 | `srcip` in `10.0.50.0/24` |

**Human-account filter (100160 / 100161):** the username must **not** match machine accounts (`*$`), `SYSTEM`, `LOCAL SERVICE`, `NETWORK SERVICE`, `ANONYMOUS LOGON`, `-`, `DWM-*` or `UMFD-*`. New domain users are covered automatically.

### Design decisions

| Decision | Reason |
|---|---|
| Windows brute-force threshold is **4**, not 5 | Domain lockout threshold is 5. Alerting at 4 means the SOC sees the attack before AD locks the account |
| Custom frequency rules 100162 / 100167 | A custom child rule takes over the event from its built-in parent, which stops Wazuh's built-in brute-force correlation from counting those failures. Our own frequency rules replace it |
| Linux brute force correlates on **source IP**; Windows on **username** | Linux SSH guessing comes from one source; Windows failures are tracked per account to line up with AD lockout |
| ATTACKER-VLAN logons are their own level-12 rules (100168/100169) | Any successful login from the attacker segment is malicious by definition, so it alerts on its own rather than relying on someone watching the dashboard. Two rules are needed because one rule can't OR across the Windows and Linux IP fields |
| Linux logons come from **journald**, not `/var/log/auth.log` | vuln-docker-01's agent reads journald; sshd, PAM and sudo all log there |

### Validation (all on live traffic, 2026-11-08)

| Rule | Test | Result |
|---|---|---|
| 100160 | `jcarter` sign-in on WS01 | ✅ Fired. Also confirmed no machine or system accounts after the exclusion change |
| 100161 | One wrong password for `jcarter` | ✅ 1 alert |
| 100162 | 4 wrong passwords for `jcarter` | ✅ Fired on the 4th, no lockout |
| 100163 | `runas /user:lab\svc-sql cmd` | ✅ 1 alert |
| 100164 | SSH login from the management PC and from Kali | ✅ Exactly 1 alert per login |
| 100165 | One wrong SSH password | ✅ Exactly 1 alert |
| 100166 | `sudo id` | ✅ User and command captured |
| 100167 | 5 wrong SSH passwords across two connections | ✅ Fired on the 5th |
| 100169 | SSH login from Kali (10.0.50.10) to vuln-docker-01 | ✅ 1 alert |
| 100168 | — | Not yet tested. ATTACKER → CORPORATE is blocked; to be validated in Scenario 04 with the temporary firewall rule |

## 100170–100177: Sensitive file access and credential dumping

These feed the **SOC — Sensitive File Access** dashboard. The custom rules share the group `sensitive_file`, but the dashboard selects by rule ID because it also uses the built-ins `92026` and `92900`.

### Log sources

| Platform | Source | Setup |
|---|---|---|
| Linux (vuln-docker-01) | auditd, read by the agent from `/var/log/audit/audit.log` | Watches persisted in `/etc/audit/rules.d/50-sensitive-files.rules`: `/etc/shadow`, `/etc/gshadow`, `/etc/sudoers`, `/etc/sudoers.d`, `/root/.ssh`, `/home/docker_user01/.ssh`, `/etc/crontab`, `/etc/cron.d`, the Docker CLI binary (execute) and the `mount` syscall |
| Windows | Sysmon (sysmon-modular) events 1 and 10, plus Security 4688 with command-line auditing | Already collected by the agents on DC01/02 and WS01/02 |

### Parents and logic

| Rule | Parent | Logic |
|---|---|---|
| 100170 | auditd (`80700`) | `audit.key` = `sens_shadow`, and `audit.exe` is **not** one of the expected readers: sudo, su, sshd, sshd-session, login, unix_chkpwd, passwd, chage, chpasswd, cron, useradd, usermod, `/usr/lib/systemd/systemd`, `/var/ossec/bin/wazuh*` |
| 100171 | auditd (`80700`) | `audit.key` matches `sens_(priv\|ssh\|persist)` |
| 100172 | auditd | `audit.key` = `sens_docker_cli` |
| 100173 | auditd (`80700`) | `audit.key` = `sens_mount`, and `audit.exe` is **not** runc, dockerd, containerd, containerd-shim-runc-v2 or `/usr/lib/systemd/systemd` |
| 100174 | `61603` (Sysmon event 1) | `commandLine` matches `vssadmin create shadow`, `esentutl … /y`, or `ntdsutil` with `ifm`, `ac i ntds` or `snapshot` |
| 100175 | `67027` (Security 4688) | `ntdsutil` with the same arguments |
| 100176 | `92025` (reg.exe) | `commandLine` matches `save HKLM\SYSTEM` or `save HKLM\SECURITY`. Sits beside the built-in `92026`, which only matches `SAM` |
| 100177 | group `sysmon_event_10` | `targetImage` is lsass, `grantedAccess` is one of `0x1fffff`, `0x1f3fff`, `0x1f0fff`, `0x1410`, `0x1438`, `0x143a`, `0x1038`, and the source isn't `MsMpEng.exe` or `wmiprvse.exe`. The built-in `92900` covers `0x1010` and `0x40`, so the two never overlap |

### Design decisions

| Decision | Reason |
|---|---|
| Windows rules anchor on the **leaf** rule (`61603`, `92025`, `67027`, `sysmon_event_10`), never on `60004` | Under the Sysmon channel rule `60004`, the built-in `61600` is a level-0 catch-all that matches every `INFORMATION` event, and Wazuh uses the first matching sibling, so a rule parented on `60004` is never reached |
| Command-line regexes use `[\\"]*` between the executable and its arguments | Wazuh stores Windows command lines with doubled backslashes and a backslash before each quote (`\"C:\\WINDOWS\\system32\\x.exe\" args`). A pattern expecting `.exe"` never matches |
| `100176` is a sibling of the built-in `92026`, not an edit of it | Built-in rules stay untouched; `92026` already covers `SAM` at level 14 |
| `100175` covers `ntdsutil` only, from 4688 | sysmon-modular doesn't log `ntdsutil` process creation, so 4688 is the only source. Limiting it to one tool avoids double alerts on hosts where Sysmon and 4688 both fire |
| The Docker socket file watch was replaced by a watch on the Docker CLI binary and the `mount` syscall | A file watch on a unix socket doesn't fire on connect |
| Exclusions were tuned from alert data, per rule | 392 of the 393 hits on `100173` came from systemd service-sandbox mounts, and `100170` fired on systemd and a Wazuh binary. Excluding those exes removed the noise without raising any threshold |
| Level 12 for credential-store reads and dumps, 10 for persistence file changes, 7 for the Docker CLI | Reading a credential store or dumping credentials is high-confidence; persistence and CLI use need context |

### Validation (live traffic, 2026-11-15)

| Rule | Test | Result |
|---|---|---|
| 100170 | `sudo cat /etc/shadow > /dev/null` on vuln-docker-01 | ✅ Fired for `/usr/bin/cat`. After the exclusions, no new systemd or Wazuh-binary alerts appeared in the checks that followed |
| 100171, 100172 | Validated earlier in the sensitive-file build | ✅ |
| 100173 | `mount -t tmpfs none /tmp/mt` on vuln-docker-01 | ✅ Fired for `/usr/bin/mount`. No new systemd alerts after tuning |
| 92026 (built-in) | `reg save HKLM\SAM` on WS01 | ✅ Fired at level 14, so this command was already covered out of the box |
| 100174 | `esentutl /y C:\Windows\win.ini /d …` and `vssadmin create shadow /for=C:` on WS01 | ✅ Both fired |
| 100175 | `ntdsutil "ac i ntds" quit quit` on DC01 | ✅ Fired. Sysmon did not log this process at all; only 4688 did |
| 100176 | `reg save HKLM\SYSTEM` and `HKLM\SECURITY` on WS01 and DC01 | ✅ Four alerts. On DC01 the commands ran as an admin and may have written hive files, so any `.hiv` files were deleted afterwards |
| 100177 | Listing lsass modules from an elevated PowerShell | ⚠️ Sysmon logged no event 10 at all, so the rule is **unvalidated on live traffic**. To be validated in Scenario 04 with a real dump technique |

### Limitations

| Limitation | Impact |
|---|---|
| No Windows file-read detection (event 4663) | Planned as `100178`. File System auditing is "No Auditing", the agent's `<query>` excludes 4663, and no SACLs exist |
| Command-based rules match tool names | A renamed binary, `comsvcs.dll` MiniDump, or shadow copies created through WMI or PowerShell evade `100174`–`100176` |
| No DCSync rule (event 4662) | Directory replication abuse is not detected |
| sysmon-modular doesn't log `ntdsutil` or benign lsass handle opens | `100175` depends on 4688 command-line auditing; `100177` has never fired live |
| systemd and Wazuh exclusions in `100170`/`100173` | A malicious systemd unit, or a process masquerading as `/var/ossec/bin/wazuh*` (which needs root), would be missed |
| `100173` still alerts on loop-device mounts by `/usr/bin/mount` with auid unset | Probably snapd; a candidate for tuning |
| Linux coverage is vuln-docker-01 only | The only Linux host with an agent |

---

## Decoders

| Decoder | Purpose |
|---|---|
| `pfsense-suricata` | Parses pfSense Suricata fast-log (format has no hostname field) |
| `pfsense-filterlog` | Parses pfSense filterlog CSV |

---

## Working with rules

All commands run on Wazuh (101).

| Task | Command |
|---|---|
| Back up before editing | `sudo cp /var/ossec/etc/rules/local_rules.xml /var/ossec/etc/rules/local_rules.xml.bak-$(date +%F-%H%M)` |
| Validate | `sudo /var/ossec/bin/wazuh-analysisd -t` |
| Apply (only after a clean validation) | `sudo systemctl restart wazuh-manager` |
| Test a log line | `sudo /var/ossec/bin/wazuh-logtest`. **Not for Windows EventChannel rules:** a pasted JSON line decodes as plain `json`, so the Windows rule chain never runs (no Phase 3). Validate those on live traffic |
| Search raw events, including level-0 | `sudo grep '<pattern>' /var/ossec/logs/archives/archives.json \| jq` |
| Confirm a rule fired | `sudo grep '"id":"<ID>"' /var/ossec/logs/alerts/alerts.json \| jq -c '{ts:.timestamp, a:.agent.name}'`. A missing `rule` field in `archives.json` is not proof that nothing matched |

> **Archives note:** `logall_json` was off from 2026-10-31 to 2026-11-08, so `archives.json` held stale data during that window. It's now enabled. Check the last timestamp before trusting an archive search: `sudo tail -1 /var/ossec/logs/archives/archives.json | jq -r .timestamp`.
