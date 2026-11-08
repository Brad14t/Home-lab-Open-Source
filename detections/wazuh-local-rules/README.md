# Wazuh Local Rules & Decoders

All custom detection content on the Wazuh manager (`soc-wazuh-01`, 10.0.20.10).

| File on the manager | Contents |
|---|---|
| `/var/ossec/etc/rules/local_rules.xml` | Custom rules `100100`–`100169` |
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

**Next free ID:** `100170` (reserved for the sensitive file access block).

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
| Test a log line | `sudo /var/ossec/bin/wazuh-logtest` |
| Search raw events, including level-0 | `sudo grep '<pattern>' /var/ossec/logs/archives/archives.json \| jq` |

> **Archives note:** `logall_json` was off from 2026-10-31 to 2026-11-08, so `archives.json` held stale data during that window. It's now enabled. Check the last timestamp before trusting an archive search: `sudo tail -1 /var/ossec/logs/archives/archives.json | jq -r .timestamp`.
