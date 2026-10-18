# My Process

<img width="385" height="113" alt="image" src="https://github.com/user-attachments/assets/e76abeac-6f1e-4edb-9292-9fbaea3ed84e" />

VULNERABLE firewall rules

Why: Right now VULNERABLE only blocks CORPORATE. This VLAN holds deliberately broken boxes, so it should be default-deny. The only exceptions are the docker host’s internet access during the build, and later its Wazuh agent traffic.

First, create a host alias in Firewall > Aliases: VULN_DOCKER = 10.0.40.10.

<img width="244" height="226" alt="image" src="https://github.com/user-attachments/assets/c312b1bf-ca1a-407d-a49d-89f0a5c9d237" />

# New Firewall Rules

## VULNERABLE VLAN Firewall Rules (10.0.40.0/24)

The VULNERABLE VLAN holds deliberately insecure systems (DVWA, bWAPP, Metasploitable 2), so it runs **default-deny**. Only DNS, gateway ping, Wazuh agent traffic, and reverse shells to the ATTACKER VLAN are allowed out. Internet access is granted to the Docker host only while it is being built, and is disabled afterward.

### Aliases

| Alias | Type | Members | Purpose |
|---|---|---|---|
| `RFC1918` | Network(s) | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | All private ranges |
| `VULN_DOCKER` | Host(s) | 10.0.40.10 | Docker host (DVWA, bWAPP, Portainer) |
| `VULN_MGMT_PORTS` | Port(s) | 22, 9443, 8080, 8081 | SSH, Portainer, DVWA, bWAPP |

### VULNERABLE interface rules

Rules are processed top to bottom; first match wins. Anything not matched hits pfSense's implicit deny.

| # | Action | Protocol | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | VULNERABLE subnets | VULNERABLE address | 53 | No | DNS to pfSense |
| 2 | Pass | ICMP (echo request) | VULNERABLE subnets | VULNERABLE address | n/a | No | Ping gateway |
| 3 | Pass | TCP | VULN_DOCKER | 10.0.20.10 | 1514-1515 | No | Wazuh agent |
| 4 | Block | Any | VULNERABLE subnets | This Firewall | any | Yes | Protect pfSense GUI |
| 5 | Pass | Any | VULNERABLE subnets | ATTACKER subnets | any | No | Reverse shells to attacker VLAN |
| 6 | Block | Any | VULNERABLE subnets | RFC1918 | any | Yes | Block other VLANs and home network |
| 7 | Pass | Any | VULN_DOCKER | any | any | No | **Temporary:** internet for build (disabled after image pulls) |


### Design notes

- **Default-deny:** vulnerable hosts can't reach CORPORATE, SECURITY (except the Wazuh agent port), the home network, or the pfSense GUI.
- **Least privilege on internet access:** rule 7 is scoped to the Docker host only. Metasploitable 2 never has internet access.
- **Temporary rule:** rule 7 is disabled once container images are pulled, leaving the VLAN fully isolated.
- **Logging on block rules:** rules 4 and 6 log, so lateral movement attempts from a compromised host show up in the firewall logs.
- **Attack path preserved:** rule 5 lets exploited hosts call back to Kali in the ATTACKER VLAN, which is needed for realistic purple-team scenarios.

# Create VM 130 Vuln Docker 

Easy command to create VM

`qm create 130 --name vuln-docker-01 --ostype l26 --cores 2 --cpu host --memory 3072 --balloon 0 --scsihw virtio-scsi-single --scsi0 lab-storage:40,format=qcow2,discard=on,ssd=1 --net0 virtio,bridge=vmbr30,firewall=0 --ide2 local:iso/ubuntu-24.04.3-live-server-amd64.iso,media=cdrom --boot order='scsi0;ide2' --agent enabled=1`

Then start it

`qm start 130`

<img width="456" height="166" alt="image" src="https://github.com/user-attachments/assets/d08e68ed-a244-4511-9aa6-0e594dceb09a" />

<img width="341" height="259" alt="image" src="https://github.com/user-attachments/assets/a908d17b-ef9e-417b-a43c-5cb464ffd055" />

<img width="491" height="130" alt="image" src="https://github.com/user-attachments/assets/9ab7af1d-c16e-4ffe-8f2b-881de86b2178" />

<img width="337" height="114" alt="image" src="https://github.com/user-attachments/assets/ad2ae1f7-9056-469d-95f7-2f132bde4326" />

<img width="283" height="235" alt="image" src="https://github.com/user-attachments/assets/bc67a9d5-dde0-4732-8960-9cd0e984e5bb" />

<img width="357" height="126" alt="image" src="https://github.com/user-attachments/assets/a64ee7d4-41e8-4c9c-8f28-1c19f6736c30" />

After the first boot, confirm outbound access and install the guest agent:

`ping -c 3 download.docker.com`

`sudo apt update && sudo apt install -y qemu-guest-agent && sudo systemctl enable --now qemu-guest-agent`

Then create a snapshot

# Install Docker Engine

`sudo apt update && sudo apt upgrade -y`

`sudo apt install -y ca-certificates curl`

`sudo install -m 0755 -d /etc/apt/keyrings`

`sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc`

`sudo chmod a+r /etc/apt/keyrings/docker.asc`

`echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`

`sudo apt update`

`sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`

Add yourself to the docker group so you don’t need sudo for every command:

sudo usermod -aG docker $USER

Logout and back in for updates to take affect

`systemctl is-enabled docker`

test docker is installed correctly `docker run --rm hello-world`

<img width="328" height="247" alt="image" src="https://github.com/user-attachments/assets/f98fee1b-0a96-4391-9881-9a3eb6cb2521" />

# Portainer

Why: Portainer gives you a web UI to manage the containers and take screenshots for the repo. --restart unless-stopped brings it back after a reboot.

`docker volume create portainer_data`

`docker run -d --name portainer --restart unless-stopped -p 9443:9443 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:lts`

Confirm its listening `ss -tlnp | grep 9443`

<img width="234" height="34" alt="image" src="https://github.com/user-attachments/assets/4d373b97-3334-466e-ac5f-49fbf6ed31c5" />

# DVWA and bWAPP via Docker Compose

Why: A single compose file defines all three containers (DVWA, its MariaDB database, and bWAPP), and you can drop it straight into the GitHub repo. It’s based on DVWA’s official compose.yml

`mkdir -p ~/vuln-apps && cd ~/vuln-apps`

`nano compose.yml`

Paste this into nano

`volumes:
  dvwa:

networks:
  dvwa:

services:
  dvwa:
    image: ghcr.io/digininja/dvwa:latest
    environment:
      - DB_SERVER=db
    depends_on:
      - db
    networks:
      - dvwa
    ports:
      - 8080:80
    restart: unless-stopped

  db:
    image: docker.io/library/mariadb:10
    environment:
      - MYSQL_ROOT_PASSWORD=***
      - MYSQL_DATABASE=***
      - MYSQL_USER=****
      - MYSQL_PASSWORD=***
    volumes:
      - dvwa:/var/lib/mysql
    networks:
      - dvwa
    restart: unless-stopped

  bwapp:
    image: raesene/bwapp
    ports:
      - 8081:80
    restart: unless-stopped`

`docker compose up -d`

`docker compose ps`

`ss -tlnp | grep -E '8080|8081'`

All three containers should show Up, and both ports should be listening.

<img width="272" height="121" alt="image" src="https://github.com/user-attachments/assets/7e066e8a-b9d9-4cf2-9a03-dd4de5e637e4" />

Initialize DVWA
- Go to http://10.0.40.10:8080 and log in with the default admin / password.
- Click Create / Reset Database. You’ll get bounced back to the login page, so log in again.
- Under DVWA Security, set the level to Low for now.

<img width="405" height="474" alt="image" src="https://github.com/user-attachments/assets/b2c10b6a-7626-4490-83d5-981250d20eef" />

<img width="114" height="386" alt="image" src="https://github.com/user-attachments/assets/55a59c9e-ae3c-43ee-bac8-597fbffe85e3" />

Initialize bWAPP
- Go to http://10.0.40.10:8081/install.php and click “here” to install the database.
- Log in with the default bee / bug.

<img width="261" height="252" alt="image" src="https://github.com/user-attachments/assets/e73076ea-6a0f-4e02-9b7e-b2dd1e24ad07" />

Then disable internet access for this vlan, verify with `curl -m 5 -I https://download.docker.com`

<img width="337" height="33" alt="image" src="https://github.com/user-attachments/assets/4b81f1cb-f9d1-4730-bf46-ecbc976a3ba7" />

Then snapshot

# Wazuh agent on vuln-docker-01

Why: It sends this host’s logs (auth, syslog, and file integrity) to Wazuh, so in Phase 6 you’ll see the host side of attacks against DVWA and bWAPP.

I will be installing this then SSHing to the docker vm since it does not connect to the internet anymore

`wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.9.2-1_amd64.deb`

`scp wazuh-agent_4.9.2-1_amd64.deb <your-user>@10.0.40.10:~`

On Docker VM

`sudo WAZUH_MANAGER='10.0.20.10' WAZUH_AGENT_NAME='vuln-docker-01' dpkg -i ~/wazuh-agent_4.9.2-1_amd64.deb`

`sudo systemctl daemon-reload`

`sudo systemctl enable --now wazuh-agent`

Check its runnign `sudo systemctl status wazuh-agent`

then snapshot the wazuh manager

# Metasploitable 2 (VM 131)

On pve01 `mkdir -p /mnt/pve/lab-storage/import && cd /mnt/pve/lab-storage/import`

`wget -O metasploitable-linux-2.0.0.zip https://sourceforge.net/projects/metasploitable/files/Metasploitable2/metasploitable-linux-2.0.0.zip/download`

Confirm you got the actual zip (about 800 MB) and not a SourceForge HTML page:

`file metasploitable-linux-2.0.0.zip`

`apt install -y unzip`

`unzip metasploitable-linux-2.0.0.zip`

Create the VM and import the disk

`qm create 131 --name vuln-msf2-01 --ostype l26 --cores 1 --memory 1024 --balloon 0 --net0 e1000,bridge=vmbr30,firewall=0`

`qm importdisk 131 Metasploitable2-Linux/Metasploitable.vmdk lab-storage --format qcow2`

`qm set 131 --sata0 lab-storage:131/vm-131-disk-0.qcow2`

`qm set 131 --boot order=sata0`

`qm start 131`

Then login for the first time with defaults

<img width="556" height="294" alt="image" src="https://github.com/user-attachments/assets/d9e9cd41-cdd2-4ab2-8208-6edc1c8fe150" />

Replace the iface eth0 inet dhcp line with:

`iface eth0 inet static
address 10.0.40.20
netmask 255.255.255.0
gateway 10.0.40.1`


<img width="604" height="216" alt="image" src="https://github.com/user-attachments/assets/c9eba585-8326-4516-ae90-7017189dbb77" />

Save and exit, then set DNS

Then ping test the gateway and docker vm

`ping 10.0.40.1` and `ping 10.0.40.10`

Then these should fail to validate isolation

`ping 8.8.8.8` and `ping 10.0.30.10`

# Suricata on the VULNERABLE interface

Why: Metasploitable can’t run an agent, so network IDS is its only visibility. Suricata on VULNERABLE sees every attack that crosses pfSense into this VLAN, which includes everything Kali sends from ATTACKER in Phase 5.

traffic inside the VLAN (10.0.40.10 ↔ 10.0.40.20) never touches pfSense, so Suricata won’t see it. That’s a good writeup point: lateral movement within a segment needs host telemetry or a span/mirror port. Security Onion in Phase 6 is the fix.

Add the interface `Services > Suricata > Interfaces > Add`

Interface: VULNERABLE
Enable: checked
Description: SURICATA_VULNERABLE
Send Alerts to System Log: checked (matches SECURITY, ready for the Wazuh forwarding open item)
Block Offenders: unchecked (IDS mode only, no IPS)
Save

<img width="469" height="460" alt="image" src="https://github.com/user-attachments/assets/3c7e072f-3a28-408f-a19a-c7fd9a5efb52" />

Why: with offloading on, the NIC hands Suricata packets with incomplete checksums or merged segments, so it can drop packets or miss detections. With VirtIO NICs, disabling checksum offload is recommended for pfSense anyway.

Fix:
Go to System > Advanced > Networking.
Under Network Interfaces, check all three:
Hardware Checksum Offloading: Disable
Hardware TCP Segmentation Offloading: Disable
Hardware Large Receive Offloading: Disable
Click Save.

Reboot pfSense (Diagnostics > Reboot). The setting only takes effect after a reboot.
<img width="484" height="295" alt="image" src="https://github.com/user-attachments/assets/cbc97b10-7423-4c3c-888c-6bc6bec7b362" />

Then checked to make sure to have all 5 agents 

<img width="229" height="168" alt="image" src="https://github.com/user-attachments/assets/a132ecd7-b6ac-47ab-b739-495685b4acc3" />

Services > Suricata > Interfaces: confirm both SECURITY and VULNERABLE are listed and green.

VULNERABLE Categories tab: check these 8, then Save:
emerging-attack_response
emerging-exploit
emerging-scan
emerging-shellcode
emerging-sql
emerging-web_server
emerging-web_specific_apps
emerging-worm

Check Suricata logs 

<img width="469" height="283" alt="image" src="https://github.com/user-attachments/assets/a272ba79-70b9-438c-9076-303e1726df44" />

Firewall > Aliases > Add

- Name: VULNERABLE_NET
- Type: Network(s)
- Network: 10.0.40.0/24
- Save, then Apply

<img width="306" height="242" alt="image" src="https://github.com/user-attachments/assets/5d0a786a-0a94-4764-a2ea-f78bbfba676d" />

Services > Suricata > Pass Lists > Add

- Name: VULN_HOME_NET
- Uncheck every auto-include box: Local Networks, WAN Gateways, WAN DNS Servers, Virtual IPs, VPNs, and so on.
- Assigned Aliases: VULNERABLE_NET
- Save

<img width="405" height="376" alt="image" src="https://github.com/user-attachments/assets/1ddd5337-b2d4-4b5a-98d4-ca2138d0b298" />

Interfaces > pencil on VULNERABLE > VULNERABLE Settings, scroll to Networks Suricata Should Inspect and Protect:

- Home Net: VULN_HOME_NET
- External Net: leave at default, which means everything not in Home Net
- Save

Retest with SQL injection

Run this from cmd on the management PC. In PowerShell, type curl.exe, since plain curl is an alias for Invoke-WebRequest.

`curl "http://10.0.40.10:8080/vulnerabilities/sqli/?id=1%27%20UNION%20SELECT%20user,password%20FROM%20users--%20&Submit=Submit"`

Check Alerts (Instance: VULNERABLE). You should see an ET WEB_SERVER ... SQL Injection or UNION SELECT alert from 192.168.0.234.

<img width="481" height="283" alt="image" src="https://github.com/user-attachments/assets/f920984d-5beb-4ce5-895d-345e19b7e656" />


This concludes phase 4


















