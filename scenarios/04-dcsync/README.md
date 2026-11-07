# Scenario 04: DCSync

> One-line summary: what technique, against what, and what detection proves it.

| | |
|---|---|
| **MITRE ATT&CK** | T1003.006 — DCSync |
| **Tactic** | Credential Access |
| **Target** | lab.local domain object |
| **Attacker** | atk-kali-01 · 10.0.50.10 |
| **Detection source** | Windows 4662 (SACL on domain object) |
| **Status** | ⬜ Planned / 🔄 Running / ✅ Detected |
| **Time to detect** | [attack timestamp → alert timestamp] |

---

## Objective

What this scenario tests and why it matters. One short paragraph. What should the SOC catch, and what would it look like in a real environment?

---

## Prerequisites & Scope

The temporary access and setup required, and the boundaries that keep it isolated.

- **Temporary firewall rule:** [source → destination, ports] — enabled for this run, disabled after.
- **Target state:** [what makes the target attackable, e.g. svc-sql has an SPN and a weak password]
- **VM set booted:** [e.g. 100, 101, 110, 140]

> **Teardown reminder:** disable the temporary rule immediately after the run.

---

## Attack

Each command in its own block with a one-line note on what it does. Never paste credentials.

```
# step 1 — what it does
command here
```

```
# step 2 — what it does
command here
```

**Result:** what the attacker obtained (e.g. a Kerberos ticket hash, cracked offline).

---

## Detection

How the defense caught it.

- **Expected telemetry:** [e.g. Windows Security 4769, encryption type 0x17 (RC4)]
- **Detection rule:** [rule ID, level, link to the rule file in `/detections`]
- **Where it surfaced:** [Wazuh dashboard / Suricata alerts / TheHive case]

### Evidence

Screenshots and, where it adds something a screenshot doesn't, a redacted log excerpt.

<!-- ![Wazuh alert](evidence/01-alert.png) -->

```
# redacted log excerpt (only if it clarifies the detection)
```

---

## Analysis

What the detection got right, and any gaps.

- Did the attack generate the expected telemetry? Any variants that would have evaded it?
- Signal quality — did it stand out, or was it buried in noise?
- Time to detect, and what that depends on.

---

## Remediation & Hardening

What would prevent or reduce this in production.

- [e.g. Use gMSAs instead of static service-account passwords]
- [e.g. Enforce AES, disable RC4]
- [e.g. Alert on anomalous SPN request volume per account]

---

## References

- [MITRE ATT&CK TXXXX.XXX](https://attack.mitre.org/)
- [Any detection/technique sources used]
