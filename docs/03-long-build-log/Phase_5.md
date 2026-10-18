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

Then `sudo systemctl enable --now ssh`

`sudo apt update && sudo apt install -y qemu-guest-agent && sudo systemctl start qemu-guest-agent`

Run a couple of connection test that should pass

`ping -c 3 10.0.40.10` and `ping -c 3 8.8.8.8`

These should fail

`ping -c 3 10.0.30.10` and `ping -c 3 10.0.20.10`

<img width="274" height="238" alt="image" src="https://github.com/user-attachments/assets/b1192642-8a4e-4e2d-afd6-26b251c52dc1" />

# Caldera on Kali

Why: Caldera is MITRE’s adversary emulation platform. It runs ATT&CK-mapped attack chains through agents, which gives you repeatable attacks to test detections against in Phase 6.


Install Caldera from the Kali repo: `sudo apt install -y caldera`

this generates a secure config with random credentials: `sudo caldera`

Leave that session running and start a new SSH session into the kali box

Find the generated config: `sudo find / -name local.yml -path '*caldera*' 2>/dev/null`

Show the users section (replace the path with the one you found): `sudo grep -A6 '^users' <path-to-local.yml>`

Save the red password

Confirm the web UI is listening: `ss -tlnp | grep 8888`

<img width="235" height="46" alt="image" src="https://github.com/user-attachments/assets/ee243422-6e68-417f-aee1-e6ebb87db92b" />

Then, from a browser on the management PC, go to http://10.0.50.10:8888 and log in as red.

<img width="957" height="425" alt="image" src="https://github.com/user-attachments/assets/5cef1ff0-7811-4045-b7a5-7665487e7690" />

# Run Caldera as a service

Why: Right now Caldera dies when you close the first SSH session. A systemd service starts it on boot and restarts it if it crashes.

Confirm the Caldera binary path: `which caldera`

Create the service file: `sudo nano /etc/systemd/system/caldera.service`

Paste this:

`[Unit]
Description=MITRE Caldera adversary emulation server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/caldera
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target`

Load the new unit file: `sudo systemctl daemon-reload`

Enable at boot and start now: `sudo systemctl enable --now caldera`

Watch the startup log until “All systems ready,” then press Ctrl+C: `sudo journalctl -u caldera -f`

<img width="484" height="202" alt="image" src="https://github.com/user-attachments/assets/0274c8ea-1669-487e-ad3d-1354cd8467cf" />

Confirm the web UI port is listening: `ss -tlnp | grep 8888`

<img width="235" height="36" alt="image" src="https://github.com/user-attachments/assets/e745c976-390e-48dd-a751-a74ba6943a8c" />

Reboot test, Reboot Kali: `sudo reboot`

I had to resetup ssh from the proxmox conxole

Then confirm caldera came back on: `systemctl is-active caldera`

Then snapshot

Writeup note: Caldera runs as root here for simplicity. In production you’d run it as a dedicated service account with only the permissions it needs.

Next up is VM 141, the Nessus Essentials scanner. Before we start that, get a free activation code at tenable.com (Nessus Essentials registration). The code arrives by email.

Register for a Nexxas account and `Download by curl` once you get to the download site.

<img width="542" height="390" alt="image" src="https://github.com/user-attachments/assets/4d9c3a69-6cdf-422b-8622-c76f1636c2f8" />

# Create VM 141 (Nessus scanner)

Why: this is a dedicated Ubuntu VM for Nessus, so plugin compilation and scans don’t compete with Kali for RAM.

Create the VM on the ATTACKER bridge with the standard lab settings:

`qm create 141 --name atk-nessus-01 --ostype l26 --cores 2 --cpu host --memory 6144 --balloon 0 --scsihw virtio-scsi-single --scsi0 lab-storage:40,format=qcow2,discard=on,ssd=1 --net0 virtio,bridge=vmbr40,firewall=0 --ide2 local:iso/ubuntu-24.04.3-live-server-amd64.iso,media=cdrom --boot order='scsi0;ide2' --agent enabled=1`

Then start it.

<img width="516" height="186" alt="image" src="https://github.com/user-attachments/assets/4aedd052-6d40-45a1-be2c-afb6661866c3" />


<img width="252" height="124" alt="image" src="https://github.com/user-attachments/assets/92ce0150-2f50-4cd6-89b8-acddc3438626" />

<img width="463" height="208" alt="image" src="https://github.com/user-attachments/assets/edd4a4cb-42f4-4d5b-a412-391965576f31" />

<img width="326" height="238" alt="image" src="https://github.com/user-attachments/assets/5c4fc7e4-5a79-493b-8e72-1889c24aa236" />

<img width="413" height="154" alt="image" src="https://github.com/user-attachments/assets/d138e347-7be5-4c9b-89b4-fe6424373787" />

One in update all packages: `sudo apt update && sudo apt upgrade -y`

Install and start the Proxmox guest agent: `sudo apt install -y qemu-guest-agent && sudo systemctl enable --now qemu-guest-agent`

Confrim it can pass `ping -c 3 10.0.40.10`

Then snapshot `phase5-nessus-fresh-install`

# Install and activate Nessus Essentials

Download the Nessus installer (paste the “Download by curl” command from Tenable’s page, Linux - Ubuntu - amd64):

`curl --request GET \
  --url 'https://www.tenable.com/downloads/api/v2/pages/nessus/files/Nessus-10.12.5-ubuntu1604_amd64.deb' \
  --output 'Nessus-10.12.5-ubuntu1604_amd64.deb'`

Install the package: `sudo dpkg -i Nessus-*.deb`

Enable Nessus at boot and start it now: `sudo systemctl enable --now nessusd`

Confirm the web UI is listening on 8834: `ss -tlnp | grep 8834`

<img width="274" height="42" alt="image" src="https://github.com/user-attachments/assets/56065433-cbe6-4239-964b-fbf3a4171916" />


Activate in the browser (management PC)
Go to https://10.0.50.11:8834 and accept the self-signed certificate warning.
Choose Register for Nessus Essentials, skip the registration form, and enter the activation code from your email.
Create the admin user and store the password in your password manager.
Plugin download and compile: this takes 20-60 minutes. Don’t close the page, restart the service, or reboot until the dashboard loads.


Once in:

<img width="665" height="370" alt="image" src="https://github.com/user-attachments/assets/79ddfa8c-94f9-43a2-8e4d-81c3ee84b52a" />

Create the scan
Close the welcome dialog with the X.
Go to Scans > New Scan > Basic Network Scan.
Fill in:
Name: Phase5 - VULNERABLE baseline
Description: Uncredentialed baseline of vulnerable VLAN
Targets: 10.0.40.10, 10.0.40.20
Click Save, then click Launch (the ▶ icon) on the scan.

<img width="321" height="268" alt="image" src="https://github.com/user-attachments/assets/0d2bfe28-6c5d-4bac-8d38-2345b8203289" />

<img width="474" height="321" alt="image" src="https://github.com/user-attachments/assets/a42a4add-7013-4c9c-96b8-a2ff4ea210cb" />

<img width="861" height="115" alt="image" src="https://github.com/user-attachments/assets/bfb7eb75-f3c4-4075-89c1-5293f4477935" />

Watch the blue side while it runs

Why: this is your first end-to-end test. A real attack tool crosses the firewall, and the defensive tools should see it.

pfSense > Services > Suricata > Alerts (Instance: VULNERABLE): expect scan and exploit-probe alerts with source 10.0.50.11.
Wazuh > agent 005 (vuln-docker-01): you may see web, auth, or connection-related events from the probing.

<img width="460" height="425" alt="image" src="https://github.com/user-attachments/assets/eb344738-c6cb-4dda-ad1c-160c99b94c5d" />

<img width="747" height="365" alt="image" src="https://github.com/user-attachments/assets/525814b6-2e56-45e3-9929-00b30f4b38f4" />

What to expect in the results
- 10.0.40.20 (Metasploitable 2): a long list of criticals, such as the vsftpd 2.3.4 backdoor, UnrealIRCd backdoor, distcc, Samba, and an outdated SSL/SSH config. That’s all expected.
- 10.0.40.10 (docker host): far fewer findings, since it’s a patched Ubuntu host. Most findings will come from the DVWA and bWAPP web services on 8080 and 8081.

<img width="846" height="252" alt="image" src="https://github.com/user-attachments/assets/c206c611-6da9-4432-b154-579033134a01" />

<img width="565" height="364" alt="image" src="https://github.com/user-attachments/assets/ba63ca9d-3ed2-43cb-a996-8f6ba839e7bb" />

This concludes phase 5.






