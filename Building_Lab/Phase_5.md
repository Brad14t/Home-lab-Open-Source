# My Process

# Attacker VLAN firewall rules

<img width="298" height="127" alt="image" src="https://github.com/user-attachments/assets/8d78430c-f82b-4de5-bb9f-8dc70d2df0c0" />

Why this layout:

- Caldera on Kali: Kali packages Caldera in its repos, so you avoid a third VM. It only needs to be reachable by its agents, which comes in Phase 6.
- Nessus on its own VM: plugin compilation is RAM- and CPU-heavy and would bog Kali down. A dedicated scanner is also how it’s deployed in real environments.
- Running set for Phase 5: 100, 101, 130, 131, 140, 141 = about 24 GB. AD VMs stay off.

## ATTACKER VLAN Firewall Rules

The ATTACKER VLAN hosts the offensive tooling (Kali with Caldera, Nessus Essentials). Unlike the VULNERABLE VLAN, it has internet access for tool updates, Kali packages, and Nessus plugin feeds. Internally it can reach **only** the VULNERABLE VLAN. CORPORATE, SECURITY, the home network, and the pfSense GUI are blocked.

### Aliases

| Alias | Type | Members | Purpose |
|---|---|---|---|
| `RFC1918` | Network(s) | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | All private ranges |
| `ATTACKER_HOSTS` | Host(s) | 10.0.50.10, 10.0.50.11 | Kali (with Caldera) and Nessus |
| `ATK_MGMT_PORTS` | Port(s) | 22, 8834, 8888 | SSH, Nessus, Caldera |

### ATTACKER interface rules

Rules are processed top to bottom; first match wins. Anything not matched hits pfSense's implicit deny.

| # | Action | Protocol | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | ATTACKER subnets | ATTACKER address | 53 | No | DNS to pfSense |
| 2 | Pass | ICMP (echo request) | ATTACKER subnets | ATTACKER address | n/a | No | Ping gateway |
| 3 | Block | Any | ATTACKER subnets | This Firewall | any | Yes | Protect pfSense GUI |
| 4 | Pass | Any | ATTACKER subnets | VULNERABLE subnets | any | No | Attack targets |
| 5 | Block | Any | ATTACKER subnets | RFC1918 | any | Yes | Block CORPORATE, SECURITY, home network |
| 6 | Pass | Any | ATTACKER subnets | any | any | No | Internet (updates, Nessus plugins) |

### WAN rule (management access)

The home network sits on pfSense's WAN, which denies inbound by default. Access is scoped to the single management PC.

| Action | Protocol | Source | Destination | Port | Log | Purpose |
|---|---|---|---|---|---|---|
| Pass | TCP | 192.168.0.234 | ATTACKER_HOSTS | ATK_MGMT_PORTS | No | SSH, Nessus, Caldera |

### Design notes

- **Scoped attack surface:** rule 4 is the only internal path out of this VLAN, so attacks can only land on intentionally vulnerable targets.
- **Internet allowed, private ranges denied:** rule 5 sits above the internet rule, so "any" in rule 6 effectively means public internet only.
- **Logging on block rules:** rules 3 and 5 log, so any attempt to reach other segments is visible.
- **IDS visibility:** attacks from this VLAN cross pfSense into VULNERABLE and are inspected by the Suricata instance on that interface. VULNERABLE's HOME_NET is scoped to 10.0.40.0/24, so ATTACKER traffic is treated as external.
- **Planned change (Phase 6):** AD attack scenarios (Kerberoasting, AS-REP roasting) and Caldera agent callbacks will need narrow ATTACKER ↔ CORPORATE rules, scoped to specific hosts and ports and added only for the exercise.

Creating an alias for the attacker vms, then apply firewall changes

<img width="208" height="262" alt="image" src="https://github.com/user-attachments/assets/0dec8b46-74f6-428c-b861-7a2306d3d413" />

# Management access

Why: You need SSH to both boxes, plus the Nessus (8834) and Caldera (8888) web UIs, from the management PC only.

Firewall > Aliases > Add: ATK_MGMT_PORTS (Port(s)) = 22, 8834, 8888

Then add a new rule passing TCP from management pc to ATTACKER_HOSTS through these above ports

Then add the new rule `route -p add 10.0.50.0 mask 255.255.255.0 192.168.0.254`

Then download the Kali ISO from Proxmox 

Get the check sum and download from `https://cdimage.kali.org/current/`

# Kali VM creation

`qm create 140 --name atk-kali-01 --ostype l26 --cores 2 --cpu host --memory 4096 --balloon 0 --scsihw virtio-scsi-single --scsi0 lab-storage:80,format=qcow2,discard=on,ssd=1 --net0 virtio,bridge=vmbr40,firewall=0 --ide2 local:iso/kali-linux-2026.2-installer-amd64.iso,media=cdrom --boot order='scsi0;ide2' --agent enabled=1`

Then start it up

<img width="541" height="279" alt="image" src="https://github.com/user-attachments/assets/e3c1365d-b089-439a-a992-e90128732771" />


First check the network

<img width="199" height="72" alt="image" src="https://github.com/user-attachments/assets/5f2ef05b-b0aa-4237-8470-4e305656be17" />

Configure the static IP in the interfaces file `sudo nano /etc/network/interfaces`

<img width="247" height="142" alt="image" src="https://github.com/user-attachments/assets/5ce068e1-9231-41d9-a185-ea3555211eef" />

`echo "nameserver 10.0.50.1" | sudo tee /etc/resolv.conf`

`sudo systemctl restart networking`

<img width="262" height="59" alt="image" src="https://github.com/user-attachments/assets/67eeb326-ea5c-4154-b7e6-fdf7b2453be1" />













