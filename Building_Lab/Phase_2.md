# My Process 

#  Installing Wazuh

I already have the ISO ready to go. So I go to `Datacenter` -> Right click `pve01` -> `Add VM`

<img width="360" height="117" alt="image" src="https://github.com/user-attachments/assets/9d07f911-f808-4234-ab47-c1aad7299b44" />

<img width="360" height="139" alt="image" src="https://github.com/user-attachments/assets/4425153d-be8f-4181-a4c7-b7b09feb01a0" />

<img width="358" height="114" alt="image" src="https://github.com/user-attachments/assets/46ffdf2c-1635-49cd-a51e-e152f8d59488" />

<img width="359" height="132" alt="image" src="https://github.com/user-attachments/assets/fca735c2-c8ea-4c50-a0f3-17c72a223101" />

<img width="364" height="79" alt="image" src="https://github.com/user-attachments/assets/75390947-51e7-4d1c-87b0-9cae11976b20" />

<img width="264" height="76" alt="image" src="https://github.com/user-attachments/assets/c6a6a4c8-c950-40a9-8654-5cccfdead696" />

<img width="361" height="104" alt="image" src="https://github.com/user-attachments/assets/d1aa80c7-a2e0-407f-9c8e-4d1d61a3f781" />

# Ubuntu Setup

<img width="456" height="142" alt="image" src="https://github.com/user-attachments/assets/fd0294f5-a367-41e2-b1b5-8c9ded8a72fd" />

<img width="391" height="329" alt="image" src="https://github.com/user-attachments/assets/585de414-27ef-48b5-ab14-82b1005beee9" />

<img width="459" height="73" alt="image" src="https://github.com/user-attachments/assets/a845b31f-9d9f-4dde-9000-225e7ea8cf3c" />

I got an error on the mirror test, I will continue with install then move to fixing DNS down the line.

<img width="447" height="322" alt="image" src="https://github.com/user-attachments/assets/e1a0a16c-84b6-4d83-abd1-98f7f559e7b6" />

<img width="435" height="189" alt="image" src="https://github.com/user-attachments/assets/24943049-eab0-4bce-8308-9799d721fc21" />

<img width="302" height="198" alt="image" src="https://github.com/user-attachments/assets/220c8ffe-7714-48a7-bf6a-d8a3a0048c40" />

<img width="427" height="129" alt="image" src="https://github.com/user-attachments/assets/a84a53a3-3e39-4ac3-bf5b-f24a01657d20" />

Then removed the media

<img width="597" height="289" alt="image" src="https://github.com/user-attachments/assets/3ab0ac0f-4463-4065-aff2-c3e94b18ca63" />

# DNS Clean up

First I try to ping to my pfsense vlan `ping -c 3 10.0.20.1`

<img width="267" height="76" alt="image" src="https://github.com/user-attachments/assets/64f541ad-316e-4deb-abb5-89f23b9dacaa" />

Try outbount connection with `ping -c 3 1.1.1.1`

<img width="265" height="58" alt="image" src="https://github.com/user-attachments/assets/742cb655-9345-494f-aa05-9a73dabb0459" />

`Troubleshooting`

First confirm Vm has the static IP I assigned of `10.0.20.10/24` on `ens18` with command `ip addr show`

<img width="372" height="96" alt="image" src="https://github.com/user-attachments/assets/a3ffed0e-db04-46be-842a-59f6e043c459" />

And checked `ip route` and both are showing correctly.

Then double checked the security Vlan is used, which it is.

Then I double checked within pfsense the security vlan is setup correctly. 

<img width="172" height="114" alt="image" src="https://github.com/user-attachments/assets/0dea9a45-6ba0-4d54-a51f-13d3fabc0a9f" />

Next I think the `firewall=1` could be the issue, maybe Proxmox added a firewall. Currently in the firewall setting, nothing is configured

<img width="354" height="208" alt="image" src="https://github.com/user-attachments/assets/a02fbc81-51c2-40f5-9c7c-b616f17f150e" />

I will disable the firewall setting and try again.

<img width="639" height="291" alt="image" src="https://github.com/user-attachments/assets/a490256c-57f0-42f9-b07d-e01a569fefb2" />

My pings this time is successful.

<img width="316" height="189" alt="image" src="https://github.com/user-attachments/assets/8a674a1e-9198-464f-9a73-8ab897129e58" />

# Wazuh Install

First command within the Ubuntu VM `curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh && sudo bash ./wazuh-install.sh -a`

Installation was complete. I try connecting to the Web ui, but was unsuccessful. So I will need to create a bridge.

I tried adding a local firewall rule allowing this traffic to go through the pfsense vlan instead of my home router, but that didnt work.

I then tried pinging from my management machine to the security vlan and wazuh, both received request timed out. 

I couldnt figure it out so I asked Claude for assistance and it found this: "Found it — this is a pfSense architecture issue, not routing or Windows.

The problem: in pfSense, your interface labeled WAN (vtnet0, 192.168.0.x) is treated as an untrusted external interface by default — even though you're using it as your home management network. pfSense auto-applies a default-deny-all inbound rule on WAN, same as it would for a real internet-facing connection. That's why traffic from your PC never even gets forwarded into the SECURITY VLAN — pfSense is dropping it right at the door.

This is actually expected/correct pfSense behavior, just not obvious until you hit it.

Fix — add an explicit WAN firewall rule:"

Trying the fix: First I went to pfsense web ui, firewall, rules, WAN, selected add below, then added it so it only applies to a single Ip address and over a single port for best security as of now.

<img width="475" height="442" alt="image" src="https://github.com/user-attachments/assets/9627edbb-3e02-4807-8137-a2db75b3ffc6" />

This didnt resolve it either. 

What the issue was, was I had set the source rule in that latest addition to /32 and it needed /24. I am now able to access my Wazuh dashboard 

<img width="291" height="244" alt="image" src="https://github.com/user-attachments/assets/dace651b-5551-47f3-b497-67c28065b7ff" />

Confirmed all Wazuh components are working

<img width="323" height="57" alt="image" src="https://github.com/user-attachments/assets/c8d680f1-5ede-4a3a-bc95-960939b6ec50" />

<img width="343" height="54" alt="image" src="https://github.com/user-attachments/assets/89dc91d1-8a0d-4f55-8871-f4d6ccc77702" />

<img width="320" height="36" alt="image" src="https://github.com/user-attachments/assets/a8917b32-7e6e-45d3-8ddc-b951fe3cf7e9" />

Then I will take a snapshot for a clean restore point

<img width="304" height="72" alt="image" src="https://github.com/user-attachments/assets/74a73f26-1217-4f25-a0b0-50a9adb2068f" />

# Splunk Install

Creating another VM 

<img width="362" height="114" alt="image" src="https://github.com/user-attachments/assets/ce2de0bd-47e6-4b27-b982-e97b5a7335e5" />

<img width="361" height="133" alt="image" src="https://github.com/user-attachments/assets/a7df8db4-edc9-41a6-82ae-6597d3b57f36" />

<img width="358" height="121" alt="image" src="https://github.com/user-attachments/assets/df89528a-8622-4d33-8775-dfa964eeaab7" />

<img width="358" height="139" alt="image" src="https://github.com/user-attachments/assets/2261a48d-0bb1-463f-8929-2b5f18c8e0ee" />

<img width="361" height="93" alt="image" src="https://github.com/user-attachments/assets/cc2ab90e-3871-4421-bec3-ea835b7fe8c3" />

<img width="266" height="74" alt="image" src="https://github.com/user-attachments/assets/d6e31417-5f0b-421e-aa23-89ce45c03ed8" />

<img width="364" height="117" alt="image" src="https://github.com/user-attachments/assets/bd8015cc-bf12-4693-88af-09ceaab7709f" />










