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

My COX router is using `192.168.0.1` so I am going to change pfSense to `192.168.0.254`











































































































