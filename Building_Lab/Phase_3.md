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

Rules are processed top to bottom; first match wins.

**Alias:** `RFC1918` = `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`

| # | Action | Protocol | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | CORPORATE subnets | CORPORATE address | 53 | No | DNS to pfSense |
| 2 | Pass | ICMP (echo request) | CORPORATE subnets | CORPORATE address | n/a | No | Ping gateway |
| 3 | Pass | TCP | CORPORATE subnets | 10.0.20.10 | 1514-1515 | No | Wazuh agents |
| 4 | Block | Any | CORPORATE subnets | This Firewall | any | Yes | Protect pfSense GUI |
| 5 | Block | Any | CORPORATE subnets | RFC1918 | any | Yes | Block other VLANs and home net |
| 6 | Pass | Any | CORPORATE subnets | any | any | No | Internet |













































