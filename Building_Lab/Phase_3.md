# My Process

# Downloaded VirtIO DriversIS & Uplaod Windows Server ISO

Start in `local storage` -> `Download From URL` `URL = https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso` -> `Download`

<img width="442" height="337" alt="image" src="https://github.com/user-attachments/assets/14706f24-445a-4d7e-aeaf-7553b6242a1e" />

Then download `Windows Server 2025` from `https://www.microsoft.com/en-us/evalcenter/download-windows-server-2025`

Then upload into local storage within proxmox

<img width="202" height="156" alt="image" src="https://github.com/user-attachments/assets/b2b52c29-2841-466f-a885-08eaf25c7634" />


# Creating New Firewall Rules for Corporate VLAN

Why: A new pfSense interface has no rules, so it denies everything. DC01 needs DNS and internet for setup and updates, and the Wazuh agents later need to reach 10.0.20.10. Everything else should be blocked so CORPORATE stays isolated like the other VLANs.

### pfSense CORPORATE Interface Rules (10.0.30.0/24)

**Alias:** `RFC1918` = `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`

| # | Action | Protocol | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | CORPORATE subnets | CORPORATE address | 53 | No | DNS to pfSense |
| 2 | Pass | ICMP (echo request) | CORPORATE subnets | CORPORATE address | n/a | No | Ping gateway |
| 3 | Pass | TCP | CORPORATE subnets | 10.0.20.10 | 1514-1515 | No | Wazuh agents |
| 4 | Block | Any | CORPORATE subnets | This Firewall | any | Yes | Protect pfSense GUI |
| 5 | Block | Any | CORPORATE subnets | RFC1918 | any | Yes | Block other VLANs and home net |
| 6 | Pass | Any | CORPORATE subnets | any | any | No | Internet |


<img width="494" height="218" alt="image" src="https://github.com/user-attachments/assets/11c64e3e-2ba4-432e-9b92-a267303d5162" />

# Create DCO1 VM

<img width="362" height="100" alt="image" src="https://github.com/user-attachments/assets/16a6ab5e-5c87-4e7f-8c02-ecebb2cbab98" />

<img width="358" height="147" alt="image" src="https://github.com/user-attachments/assets/5b0767aa-b69f-4bbd-9eb5-ac2fe1e214fd" />

<img width="358" height="160" alt="image" src="https://github.com/user-attachments/assets/38137691-efaf-405e-9c0b-7e8e4b685b1f" />

<img width="361" height="136" alt="image" src="https://github.com/user-attachments/assets/7bf7d0ec-4b60-4ef4-9b00-b752c0b20322" />

<img width="358" height="83" alt="image" src="https://github.com/user-attachments/assets/fb10343d-6231-49f2-a64d-cedbf62a8838" />

<img width="259" height="64" alt="image" src="https://github.com/user-attachments/assets/398d29bb-bba1-4c9e-878e-89332069d5ac" />

<img width="363" height="106" alt="image" src="https://github.com/user-attachments/assets/5ba16922-09ac-428d-bcb3-a0b9969cea71" />

<img width="304" height="232" alt="image" src="https://github.com/user-attachments/assets/0e51fe02-54a6-4dc1-898f-418c79827a45" />

<img width="301" height="228" alt="image" src="https://github.com/user-attachments/assets/65c1bb5b-bdec-4779-9680-adaf8ca8973f" />

<img width="291" height="226" alt="image" src="https://github.com/user-attachments/assets/39b57e0f-c8ee-4ce3-ae5e-b79401f426d8" />

Then I install the drivers

<img width="325" height="242" alt="image" src="https://github.com/user-attachments/assets/789741f3-37fa-40ee-a725-79c130f5adaa" />

Then I confirm with `qm agent 110 ping` within host shell (Success returns nothing; an error means the agent isn’t running.)

<img width="148" height="27" alt="image" src="https://github.com/user-attachments/assets/72910dee-f70a-4fb5-bef0-41fe23c12bc4" />

Confrim I get the correct IP with `ipconfig`

Then ping the gateway with `ping 10.0.30.1`

<img width="231" height="208" alt="image" src="https://github.com/user-attachments/assets/8e6ac47d-4974-40e1-a549-ed4a48e7c80d" />

Set DNS

<img width="414" height="199" alt="image" src="https://github.com/user-attachments/assets/14cc719c-4c91-40a9-9b5f-45f2f982d959" />

Install the rules `Install-WindowsFeature AD-Domain-Services -IncludeManagementTools`

Then create the forest and password associated `Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDns`

Next is to point Corporate DHCP to DC01

Why: Clients must use the DC for DNS, or they can’t find the domain to join it. DC01 forwards internet lookups on to pfSense.

pfSense → Services → DHCP Server → CORPORATE:

Create OU Structure

lab.local
└── LAB
    ├── Admins
    ├── Service Accounts
    ├── Groups
    ├── Servers
    ├── Workstations
    └── Users
        ├── IT
        ├── HR
        ├── Finance
        └── Sales






