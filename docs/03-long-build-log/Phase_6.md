# My Process

# TheHive Tuning 

Why: There’s already a ~1,650/day medium-alert flood; tune before adding anything new or TheHive drowns.

Boot pfSense + Wazuh

What should be running right now is: `100`, `101`, `103`, `104` 

Ensure ports 9042 and 9200 are listening on 103

Confirm Wazuh services are actually up

`sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager`

Confirm the ports are listening (1514/1515 agents, 55000 API, 9200 indexer, 443 dashboard): `sudo ss -tlnp | grep -E ':(1514|1515|55000|9200|443)\b'`

Pull the noisiest rules (level ≥ 7)

Make sure jq is present: `which jq || sudo apt-get install -y jq`

Count of level-7+ alerts in today’s file, to confirm the volume: `sudo jq -r 'select(.rule.level >= 7) | .rule.id' /var/ossec/logs/alerts/alerts.json | wc -l`

<img width="484" height="46" alt="image" src="https://github.com/user-attachments/assets/b2779be3-c0ec-4687-abdc-eb08da00d222" />

Top 30 rules by volume (count, rule ID, level, description) — this is the table we tune from: `sudo jq -r 'select(.rule.level >= 7) | "\(.rule.id)\t\(.rule.level)\t\(.rule.description)"' /var/ossec/logs/alerts/alerts.json | sort | uniq -c | sort -rn | head -30`

Apply the safe part now (dpkg → level 3)

See what’s already in the file first, so IDs don’t clash: `sudo cat /var/ossec/etc/rules/local_rules.xml`

Edit: `sudo nano /var/ossec/etc/rules/local_rules.xml`

Add this block at the end, after any existing </group>:

`<group name="local,tuning,">
  <rule id="100100" level="3">
    <if_sid>2902,2903,2904</if_sid>
    <description>Dpkg package activity (tuned down from level 7; kept for audit).</description>
  </rule>
</group>`

Validate the rules before restarting (any XML error shows here): `sudo /var/ossec/bin/wazuh-analysisd -t`

Restart the manager: `sudo systemctl restart wazuh-manager`

Confirm it came back up: `sudo systemctl status wazuh-manager --no-pager`

Test with a real dpkg line

Start logtest: `sudo /var/ossec/bin/wazuh-logtest`

Paste this line at the prompt: `2026-10-16 06:57:32 status half-configured dbus:amd64 1.14.10-4ubuntu4.1`

<img width="388" height="202" alt="image" src="https://github.com/user-attachments/assets/79386526-9a0c-44b5-beb6-80ffa32abb65" />

Rule 100100 matched at level 3, so the dpkg tuning works. The alert is still written to alerts.json and the dashboard for audit, but at level 3 it’s below the TheHive threshold of 7 and won’t be forwarded.


- Symptom: dpkg events (2902/2903/2904) were about half of all level 7+ alerts (961 of the history).
- Root cause: stock Wazuh rates package activity at level 7, which is noise during routine apt upgrades.
- Fix: child rule 100100 drops them to level 3 (kept for audit, not sent to TheHive).
- Takeaway: tune by rule, not by raising the global threshold. A higher threshold would also hide the level 7+ detections you want in Phase 6.

Add the second rule batch: `sudo nano /var/ossec/etc/rules/local_rules.xml`

Add in:
`<group name="local,tuning,">
  <rule id="100101" level="0">
    <if_sid>510</if_sid>
    <match>Trojaned version of file '/usr/bin/diff'|Trojaned version of file '/bin/diff'</match>
    <description>Rootcheck false positive: generic trojan signature matches stock diff (verified with dpkg --verify).</description>
  </rule>

  <rule id="100102" level="3">
    <if_sid>19007</if_sid>
    <description>SCA check failed (tuned down from level 7; see SCA dashboard for details).</description>
  </rule>

  <rule id="100103" level="3">
    <if_sid>550,553</if_sid>
    <field name="file">^/etc/ssl/certs/|^/boot/|^/etc/ld.so.cache$</field>
    <description>FIM change in package-managed path (certs, boot, ld.so.cache); tuned down from level 7.</description>
  </rule>
</group>`

<img width="475" height="418" alt="image" src="https://github.com/user-attachments/assets/b9608f4b-6dfd-45e1-9ff0-b064ee9df335" />


Restart 

Logtest can’t simulate rootcheck or FIM events, so verify against live behavior. Record the 510 count before the restart’s rootcheck scan finishes: `sudo grep -c '"id":"510"' /var/ossec/logs/alerts/alerts.json`

Rootcheck re-runs on manager start. Wait about 10 minutes and run the same command. The count should not increase. For the 510 check, you can also run this and expect no diff lines from today: `sudo grep '"id":"510"' /var/ossec/logs/alerts/alerts.json | grep -c diff`

The SCA and FIM overrides get verified naturally the next time apt runs or SCA rescans (every 12 hours by default). Spot-check later with: `sudo jq -r 'select(.rule.id=="100102" or .rule.id=="100103") | "\(.rule.id)\t\(.rule.level)\t\(.agent.name)"' /var/ossec/logs/alerts/alerts.json | sort | uniq -c`

Next is to snap shot proxmox host and pfsense vm

Open a syslog listener on Wazuh

On the Wazuh VM, back up the config: `sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak-phase6`

See what <remote> blocks exist (expect one secure on 1514): `sudo grep -n -A6 '<remote>' /var/ossec/etc/ossec.conf`

<img width="334" height="69" alt="image" src="https://github.com/user-attachments/assets/83a58756-dae7-41b2-b576-de610bf653d4" />

Open the config: `sudo nano /var/ossec/etc/ossec.conf`

Add this block directly after the existing </remote> (inside <ossec_config>):

`  <remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>udp</protocol>
    <allowed-ips>10.0.20.1</allowed-ips>
    <local_ip>10.0.20.10</local_ip>
  </remote>`


Why: allowed-ips limits the listener to pfSense’s SECURITY address, so nothing else on the network can inject fake alerts.


Restart the manager: `sudo systemctl restart wazuh-manager`

Confirm it’s listening on UDP 514: `sudo ss -ulnp | grep ':514'`

# Point pfSense at Wazuh

In the pfSense GUI, go to Status > System Logs > Settings, scroll to Remote Logging Options:

<img width="360" height="115" alt="image" src="https://github.com/user-attachments/assets/651e3a3b-fdbd-47ff-bd97-e342bd8a8401" />

Why Everything: Suricata alerts go out on the LOCAL1 facility, and Everything is the simplest option that includes them. It also forwards logged firewall blocks (your Block rules have logging on), which is useful for Phase 6: you’ll want to see Kali’s blocked attempts at CORPORATE. Watch the volume.

Save. No firewall rule is needed, since traffic from pfSense itself to its own SECURITY subnet isn’t filtered.

<img width="475" height="325" alt="image" src="https://github.com/user-attachments/assets/4eb27019-6265-4545-be9f-989a2744862a" />

# Confirm packets arrive

On the Wazuh VM, watch for syslog packets (should show pfSense lines within a minute):

`sudo tcpdump -i any -nn -A udp port 514 -c 5`<img width="484" height="295" alt="Screenshot 2026-10-17 134910" src="https://github.com/user-attachments/assets/575ffa73-eb31-4acf-b96f-f54a484b4848" />

# Generate a Suricata alert

Run from your management PC (PowerShell). This sends a SQL injection–style request to DVWA, crossing pfSense so the VULNERABLE instance sees it:

`curl.exe "http://10.0.40.10:8080/vulnerabilities/sqli/?id=1%27%20UNION%20SELECT%20NULL,NULL--%20-&Submit=Submit"`

`curl.exe "http://10.0.40.10:8080/vulnerabilities/sqli/?id=1%27%20UNION%20SELECT%20user,password%20FROM%20users--%20-&Submit=Submit"` 

Then repeat the capture from 2.4 and look for a line containing suricata[...] with [Classification: ...]. If it doesn’t appear immediately, rerun the curl, as the Home Net scoping needs the traffic to match.

<img width="487" height="118" alt="image" src="https://github.com/user-attachments/assets/fe4cbd79-b090-4866-9e3d-27f76d9d08aa" />

Run the line through Wazuh’s decoders

Start logtest on the Wazuh VM:

`sudo /var/ossec/bin/wazuh-logtest`

Paste this line at the prompt (reassembled from your capture, with the syslog priority stripped):

`Oct 17 14:09:46 suricata[50656]: [1:2010963:7] ET WEB_SERVER SELECT USER SQL Injection Attempt in URI [Classification: Web Application Attack] [Priority: 1] {TCP} 192.168.0.234:63744 -> 10.0.40.10:8080`

Then paste a filterlog line too, so we can see how the WAN noise is rated (reassembled from your earlier capture):

`Oct 17 13:51:20 filterlog[27223]: 4,,,1000000103,vtnet0,match,block,in,4,0xc0,,1,52201,0,none,2,igmp,28,192.168.0.1,224.0.0.1,datalength=8`

<img width="402" height="121" alt="image" src="https://github.com/user-attachments/assets/d21254e8-8ad0-4e60-bf94-e6bd088c23f6" />

The Suricata alerts already land in Wazuh, but as useless level-2 “Unknown problem” entries. They’d never reach TheHive.

The fix

Write a custom decoder that keys on the message shape instead of the program name, plus rules that map Suricata’s own priority to Wazuh levels:

# Add the decoder

Check what’s already in the file: `sudo cat /var/ossec/etc/decoders/local_decoder.xml`

Edit it: `sudo nano /var/ossec/etc/decoders/local_decoder.xml`

Append this at the end, after any existing content:

`<decoder name="pfsense-suricata">
  <prematch type="pcre2">^\[\d+:\d+:\d+\] </prematch>
  <regex type="pcre2">^(.+?) \[Classification: (.+?)\] \[Priority: (\d+)\] \{(\w+)\} (\d+\.\d+\.\d+\.\d+):(\d+) -> (\d+\.\d+\.\d+\.\d+):(\d+)</regex>
  <order>signature,classification,priority,protocol,srcip,srcport,dstip,dstport</order>
</decoder>`


<img width="488" height="289" alt="image" src="https://github.com/user-attachments/assets/597b54ae-c3d1-43ee-b7eb-5b06a338d4dd" />

Why this prematch: the [gid:sid:rev] prefix is how Suricata fast-log lines start, and with the hostname missing it’s the first thing in the message body.

Add the rules

`sudo nano /var/ossec/etc/rules/local_rules.xml`

Append after the existing groups:

`<decoder name="pfsense-suricata">
  <prematch type="pcre2">\[\d+:\d+:\d+\] .+\[Priority: \d+\]</prematch>
</decoder>

<decoder name="pfsense-suricata-fields">
  <parent>pfsense-suricata</parent>
  <regex type="pcre2">\[\d+:\d+:\d+\] (.+?) \[Classification: (.+?)\] \[Priority: (\d+)\] \{(\w+)\} (\d+\.\d+\.\d+\.\d+):(\d+) -> (\d+\.\d+\.\d+\.\d+):(\d+)</regex>
  <order>signature,classification,priority,protocol,srcip,srcport,dstip,dstport</order>
</decoder>>`

Validate, then test: `sudo /var/ossec/bin/wazuh-analysisd -t`

Rerun logtest (it loads the new decoder on start): `sudo /var/ossec/bin/wazuh-logtest`

Paste the same line:

`Oct 17 14:09:46 suricata[50656]: [1:2010963:7] ET WEB_SERVER SELECT USER SQL Injection Attempt in URI [Classification: Web Application Attack] [Priority: 1] {TCP} 192.168.0.234:63744 -> 10.0.40.10:8080`


Expected:

- Phase 2: decoder pfsense-suricata, with signature, classification, priority, srcip and the other fields filled in.
- Phase 3: rule 100112, level 8.

<img width="484" height="385" alt="image" src="https://github.com/user-attachments/assets/471b5261-be9e-46ea-b0f9-0caf28365c27" />

# Live end-to-end test

Restart the manager so real events use the new decoder and rules: `sudo systemctl restart wazuh-manager`

Confirm it’s back up and still listening on UDP 514: `sudo ss -ulnp | grep ':514'`

Trigger the alert from your management PC (PowerShell): `curl.exe "http://10.0.40.10:8080/vulnerabilities/sqli/?id=1%27%20UNION%20SELECT%20user,password%20FROM%20users--%20-&Submit=Submit"`

Confirm the real alert landed on the Wazuh VM: `sudo grep '"id":"100112"' /var/ossec/logs/alerts/alerts.json | tail -1 | jq '{level: .rule.level, desc: .rule.description, src: .data.srcip, dst: .data.dstip}'`

<img width="763" height="69" alt="image" src="https://github.com/user-attachments/assets/c26d96ef-eb4d-4aae-aaaa-64520307f68a" />

Snapshot 100, and 101

# Firewall Adjustments

Current ATTACKER ruleset blocks RFC1918 (rule 5), which covers CORPORATE, so we need an explicit pass above it, scoped tightly: Kali only, DC01 only, ports 88 and 389 only.

Create an alias called `PHASE6_AD_PORTS` (Kerberos + LDAP for Kerberoast/AS-REP roast testing) Ports `88`, `389`

Go to Firewall > Rules > ATTACKER. We need this rule to sit above rule 5 (“Block Any ATTACKER subnets → RFC1918”), so click Add with the up-arrow icon (adds above the selected rule) after selecting rule 5, or add it anywhere and drag it above rule 5 afterward.

<img width="467" height="447" alt="image" src="https://github.com/user-attachments/assets/19a778bf-6b7d-4b86-998c-261d2bc6bca5" />

Verify

From Kali (SSH or console), confirm Kerberos is reachable (expect a connection, even if the protocol handshake fails):

`nc -zv -w3 10.0.30.10 88` `nc -zv -w3 10.0.30.10 389`

<img width="258" height="94" alt="image" src="https://github.com/user-attachments/assets/81fc5b82-6e85-47b8-9beb-b69f2086cc0d" />






















