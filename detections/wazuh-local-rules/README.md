# Wazuh Local Rules & Decoders

The custom detection content added to Wazuh during Phase 6. Paste the actual
`local_rules.xml` and `local_decoder.xml` from the Wazuh manager
(`/var/ossec/etc/rules/` and `/var/ossec/etc/decoders/`) into this folder.

## Rules in this lab

| Rule ID | Level | Purpose |
|---|---|---|
| 100100 | 3 | Dpkg package activity — tuned down from 7, kept for audit |
| 100101 | 0 | Rootcheck false positive on stock `diff` (verified with `dpkg --verify`) |
| 100102 | 3 | SCA check failures — tuned down from 7 |
| 100103 | 3 | FIM changes in package-managed paths (certs, boot, ld.so.cache) |
| 100110 | 3 | pfSense Suricata alert (base) |
| 100111 | 5 | pfSense Suricata priority 2 |
| 100112 | 8 | pfSense Suricata priority 1 (HIGH — reaches case management) |
| 100120 | 2 | pfSense firewall block (base) |
| 100121 | 7 | Block from ATTACKER VLAN (probing beyond scope) |
| 100122 | 7 | Block from VULNERABLE VLAN (compromised host egress) |

## Decoders

| Decoder | Purpose |
|---|---|
| `pfsense-suricata` | Parses pfSense Suricata fast-log (no hostname field) |
| `pfsense-filterlog` | Parses pfSense firewall block (filterlog) CSV |

> Additional scenario-specific rules (Kerberoasting, Log4Shell, scan-frequency)
> are added as each scenario in [../../scenarios/](../../scenarios/) is completed.
