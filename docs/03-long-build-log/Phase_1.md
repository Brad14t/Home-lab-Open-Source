# My Process

# Identify Interfaces

I had started this lab, but need to make changes to it. This will be my process of setting up the lab.

First I use the `Shell` into the proxmox host. Then I run this command to check existing network bridges:
`ip addr show | grep vmbr`

<img width="736" height="68" alt="image" src="https://github.com/user-attachments/assets/570188c5-8d2e-4185-967a-74dce1519b07" />

Here I can see:

- `vmbr0`: 192.168.0.250/24
- `vmbr1`: (no IP assigned - state UNKNOWN)

`vmbr0` is the Management bridge.

# Create Additional Bridge

Current bridge status:

<img width="713" height="275" alt="image" src="https://github.com/user-attachments/assets/7c5dec69-c2a2-49e6-b672-2c0ea84fc584" />

Creating `vmbr10` for `Security Tools`

<img width="452" height="178" alt="image" src="https://github.com/user-attachments/assets/78b867fb-c437-403a-b9fe-832f021ca593" />

Creating `vmbr20` for `Corporate Network VLAN`

<img width="447" height="178" alt="image" src="https://github.com/user-attachments/assets/b3b34e44-af4c-44fd-8f4e-2627fcea493c" />

Creating `vmbr30` for `Vulnerable Apps VLAN` & `vmbr40` for `Attacker Network VLAN`

<img width="495" height="170" alt="image" src="https://github.com/user-attachments/assets/5361bbcb-a8d0-4924-8b41-807bb96d6274" />

# Configure pfSense VM

Adding network devices

`net2` - Security Tools

Bridge vmbr10, No for firewall because I want pfSense handeling the rules, Model as VirtIO because it offers the best performance for virtual networks. 

<img width="446" height="125" alt="image" src="https://github.com/user-attachments/assets/7dce395a-d020-4e3f-9335-dad8341cd03e" />

`net3` - Corporate

<img width="449" height="129" alt="image" src="https://github.com/user-attachments/assets/4e900672-f0ce-4cb4-8876-37ad99e05c24" />

`net4` - Vulnerable Apps

<img width="448" height="133" alt="image" src="https://github.com/user-attachments/assets/f675b5d3-dd4d-49fb-abf6-8d1e5d364a03" />

`net5` - Attacker Network

<img width="447" height="127" alt="image" src="https://github.com/user-attachments/assets/cb388955-c667-47b7-9d8f-d161ab7bea91" />

Attaching `pfSense`

<img width="395" height="278" alt="image" src="https://github.com/user-attachments/assets/b276fbfe-c782-4734-aabc-95734d71feea" />

`Current Status:`

- 6 network interfaces configured
- pfSense ISO attached to CD/DVD drive

<img width="585" height="328" alt="image" src="https://github.com/user-attachments/assets/71be00c0-440f-4a63-b204-1ca23e1723c8" />

`ERROR`

Windows 11 booted up, I think the boot order is wrong, since pfSense is what I want.

I shutdown the VM 

Checked boot order, it was correct. But I must have selected Windows 11, switched it over to pfSense ISO and it booted up as normal.

<img width="539" height="303" alt="image" src="https://github.com/user-attachments/assets/1f0bc0f8-0db1-4684-8930-4ab38193fb40" />

For the WAN option I select `vtnet0` since I dont need this for the lab.

<img width="527" height="286" alt="image" src="https://github.com/user-attachments/assets/1e1951d8-3d83-42a9-b94b-58b898896115" />

For LAN interface I select `vtnet1`

<img width="465" height="258" alt="image" src="https://github.com/user-attachments/assets/74f61be5-f4aa-45c8-aef4-dcba3a6692a7" />

Then I change the IP Address selecting `I`

<img width="480" height="272" alt="image" src="https://github.com/user-attachments/assets/80253f27-5090-4454-b0ce-61823f579ffb" />

<img width="265" height="131" alt="image" src="https://github.com/user-attachments/assets/a77566d6-57f6-456a-bd50-79df22087419" />

`ERROR`

My pfSense VM froze here, I rebooted the VM within Proxmox.

Further pfSense configurating, setting the static IP address and subnet mask.

<img width="548" height="264" alt="image" src="https://github.com/user-attachments/assets/4f73cf22-965b-4325-b398-c35e43d1dc8e" />

Select no for upstream gateway since I dont need WAN.

<img width="412" height="47" alt="image" src="https://github.com/user-attachments/assets/b8820869-efcb-4552-805e-d300a04966f9" />

And enable DHCP server on LAN to allow pfSense to assign IP addresses to devices.

Then select the DHCP range

<img width="488" height="56" alt="image" src="https://github.com/user-attachments/assets/db9b3066-ee9d-48d6-b578-42ee9d983380" />

<img width="539" height="55" alt="image" src="https://github.com/user-attachments/assets/331cfeb0-aaf0-42a6-976a-b6892a5d1cc7" />

`ERROR`

I am going to change pfSense to `192.168.0.254`

<img width="414" height="53" alt="image" src="https://github.com/user-attachments/assets/d255ee27-9ed9-42a6-bc1f-707d054dc4b6" />

# Signing into pfSense

`ERROR`

I cant access my pfSense from my web browser, so to troubleshoot I am going to create an Ubuntu vm within proxmox, and try connecting from that.

<img width="362" height="334" alt="image" src="https://github.com/user-attachments/assets/1a20f65c-df7b-429e-b342-7d5ca5386245" />

I start to create a new VM.

<img width="270" height="139" alt="image" src="https://github.com/user-attachments/assets/10d46ca2-33c0-4536-a8df-358ea0a70fbb" />

<img width="529" height="162" alt="image" src="https://github.com/user-attachments/assets/e419f51d-dd8e-4454-94fc-3a087f192fce" />

<img width="535" height="158" alt="image" src="https://github.com/user-attachments/assets/17714f5b-1cd4-4b67-9334-4fb0119d36cc" />

<img width="534" height="182" alt="image" src="https://github.com/user-attachments/assets/2a9ad502-e611-435b-a647-c8fc07c5095b" />

<img width="533" height="117" alt="image" src="https://github.com/user-attachments/assets/47bee0c0-b7fc-4481-9a2c-a65b96f3a783" />

<img width="527" height="156" alt="image" src="https://github.com/user-attachments/assets/8f67c5f1-37b4-498f-975e-eb7611f17783" />

VM has been created, I will now boot into it and try ti connect to pfSense

<img width="173" height="120" alt="image" src="https://github.com/user-attachments/assets/d4bed4df-935b-4d39-baee-d3d73297653c" />

I set up a profile, then I deselect the SSH Server. I dont need this.

<img width="607" height="199" alt="image" src="https://github.com/user-attachments/assets/38e7ea61-eacc-401a-90b8-5dd723f35d02" />

Now im in the Ubuntu VM I check my IP address with `ip addr show`

<img width="617" height="187" alt="image" src="https://github.com/user-attachments/assets/f2c9ac5b-3b82-4d5d-bab9-8fc435fb7059" />

# Testing VM

Test ping to pfSense with `ping -c 4 192.168.0.254`

`ERROR`

I got 100% packet loss, my first step in troubleshooting is to run `ip route show`. Everything looks normal, so next I will check if DHCP is configured.

<img width="476" height="58" alt="image" src="https://github.com/user-attachments/assets/41d53a19-4003-484f-b215-3751e53e5a74" />

In pfSense I open the LAN configuration

<img width="501" height="282" alt="image" src="https://github.com/user-attachments/assets/af3fa442-2f3c-443c-a746-c8c0bb2e48a0" />

<img width="495" height="150" alt="image" src="https://github.com/user-attachments/assets/0ea69bd5-36e7-42e6-941d-16fd2163d02e" />

Next I will renew IP in Ubuntu VM with `sudo ip link set ens18 down` > `sudo ip link set ens18 up` > `sleep 3` > `ip addr show ens18`

<img width="610" height="143" alt="image" src="https://github.com/user-attachments/assets/b4289ed6-c25b-4e7a-b04a-6af26562edb7" />

Now with an updated IP address of `192.168.0.150` ill try to ping pfSense again.

I get 100% packet loss again

<img width="386" height="88" alt="image" src="https://github.com/user-attachments/assets/0070157f-4d46-4ed8-9c64-c2320f485813" />

Next I will try to ping Ubuntu from pfSense, but get 100% packet loss again, this shows this isnt a one sided issue.

I will try switching pfSense over to `vmbr0`

I shut down the pfSense VM, and change that setting.

<img width="700" height="239" alt="image" src="https://github.com/user-attachments/assets/03e6fd13-375a-48ba-8e72-3710d3b46350" />

Now my WAN and LAN are on the same network, which is a problem.

<img width="427" height="57" alt="image" src="https://github.com/user-attachments/assets/d6fdf945-c80e-4423-9183-7396c8c25438" />

I remove the WAN interface since I dont need it.

Still I can ping, but I am getting host unreachable

<img width="406" height="194" alt="image" src="https://github.com/user-attachments/assets/9f3ae293-0f4a-40a9-af8f-821a267e5862" />

I am going to try pinging my pc from my pfSense. Didnt work. Next I am trying reassigning interfaces with pfSense.

Looks like I had to remove the LAN interface and change the WAN to take its place.

<img width="472" height="181" alt="image" src="https://github.com/user-attachments/assets/8c451633-b87a-4037-b47b-0f9cc922470e" />

<img width="678" height="565" alt="image" src="https://github.com/user-attachments/assets/486f23f8-1a59-4401-9f57-974a51d0d367" />

# pfSense Setup

<img width="571" height="835" alt="image" src="https://github.com/user-attachments/assets/3f077b63-85e5-4c19-91d4-b776a0cc2094" />

<img width="552" height="338" alt="image" src="https://github.com/user-attachments/assets/b423ea60-f183-4d80-8592-3ae6018e6390" />

Went back and deleted test VM.

Next I am going to create my vlan assignments:

<img width="515" height="186" alt="image" src="https://github.com/user-attachments/assets/2dea1864-01a4-44a5-adf0-94a62c320ce7" />

Confirmed within console of pfsense 

<img width="216" height="48" alt="image" src="https://github.com/user-attachments/assets/a6e6f357-ca0d-4ef3-9f3d-d1bbfa840389" />

`ERROR` I couldnt connect back to my pfsense GUI after I saved changes, I ended up finding the fix and added a rule via the shell in pfsense with this command: `easyrule pass wan tcp 192.168.0.234`

Then I set the other vlans:

`LAN` to `10.0.20.1` - `SECURITY`

`OPT1` to `10.0.30.1` - `CORPORATE`

`OPT2` to `10.0.40.1` - `Vulnerable`

`OPT3` to `10.0.50.1` - `ATTACKER`

<img width="575" height="406" alt="image" src="https://github.com/user-attachments/assets/6dbae929-f742-44d1-a811-0d1e75e52d72" />

<img width="546" height="410" alt="image" src="https://github.com/user-attachments/assets/0fce0883-35cb-4a19-b040-6d12f88d5ab6" />

<img width="558" height="400" alt="image" src="https://github.com/user-attachments/assets/b12eb729-bfe0-4608-bb1e-588016d7a7a7" />

`Enabling DHCP`

Within `Security` tab:

I enabled - `Enable DHCP server on SECURITY interface`

Added the address pool range to make room for reservations later - `10.0.20.100` to `10.0.20.200`


Within `Corporate` tab:

I enabled - `Enable DHCP server on SECURITY interface`

Added the address pool range to make room for reservations later - `10.0.30.100` to `10.0.30.200`

Within `Vulnerable` tab:

I enabled - `Enable DHCP server on SECURITY interface`

Added the address pool range to make room for reservations later - `10.0.40.100` to `10.0.40.200`

Within `Attacker` tab:

I enabled - `Enable DHCP server on SECURITY interface`

Added the address pool range to make room for reservations later - `10.0.50.100` to `10.0.50.200`

`Firewall rules`

I am isolating vul/attacker vlans from corp vlan.

How I look at it:

`SECURITY` (your tools: Wazuh, Splunk, TheHive, later replaced by DFIR-IRIS) should be able to reach into every other VLAN (monitoring needs visibility everywhere)

`CORPORATE` should NOT be reachable from `VULNERABLE` or `ATTACKER` (this is the "protect the business" zone)

`VULNERABLE` and `ATTACKER` should be able to talk to each other (attacker box attacks vulnerable box)

`VULNERABLE` and `ATTACKER` should NOT be able to reach `CORPORATE` or `SECURITY` except where explicitly needed (e.g., Wazuh agents phoning home to the Wazuh manager on SECURITY)


First firewall change I will make is for `Vulnerable` -> action will be `block` -> protocol will be `any` -> source will be `vulnerable subnets` -> destination `corporate subnets` -> description `Block Vulnerabale to Corporate`

Second firewall change I will make is for the `Vulnerable` -> action will be `pass` -> protocol will be `any` -> source will be `vulnerable subnets` -> destination `attacker subnets` -> description `Allow Vulnerable to Attacker`

Third firewall change I will make is for the `Attacker` -> action will be `block` -> protocol will be `any` -> source will be `attacker subnets` -> destination `corporate subnets` -> description `Allow Vulnerable to Attacker`

# Install and Configure Suricata IDS

Within `pfsense` I went to the `System` tab -> `Package Manager` -> Searched for `Suricata` and installed it.

After installed I go to `Services` -> `Suricata` -> `add`

<img width="461" height="384" alt="image" src="https://github.com/user-attachments/assets/66816446-1f67-48f5-b5af-fc1ec10d7060" />

Confirmed it is running:

<img width="464" height="101" alt="image" src="https://github.com/user-attachments/assets/68c03675-418f-4b61-9937-7d6b1946e938" />

Now to close out phase 1 I will create a snapshot of `pfsense`

<img width="226" height="112" alt="image" src="https://github.com/user-attachments/assets/e1f965bd-075f-4819-badd-0356bd1dee39" />

<img width="706" height="187" alt="image" src="https://github.com/user-attachments/assets/d71efcd0-d70f-4ef4-87d9-8f7a000c3ff4" />

# Additional Storage Needs

At this point I realized that I dont have enough storage within my proxmox machine. I went out and bought a 4tb SSD, and a caddy to hold it physically within the PC. I then mounted the new drive and created it so it can persist through restarts. 

<img width="442" height="55" alt="image" src="https://github.com/user-attachments/assets/67d348d0-303c-4bea-a18b-2e9c0bfadbad" />

# Phase 2 is ready to begin!





















