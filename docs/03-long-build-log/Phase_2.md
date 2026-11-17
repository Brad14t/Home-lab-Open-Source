# My Process 

> **Note:** this log records the build as it happened. Case management was originally TheHive + Cortex; it was later replaced by DFIR-IRIS on the same VMs (103/104), and DFIR-IRIS is what the scenarios use. References to TheHive below are kept as the historical record.

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

# Ubuntu Install

<img width="444" height="146" alt="image" src="https://github.com/user-attachments/assets/1dc54337-70a9-4475-80a0-c3af47803ae1" />

<img width="369" height="276" alt="image" src="https://github.com/user-attachments/assets/deb33b54-416e-43ec-9ee8-9896ebcd0c1f" />

<img width="456" height="84" alt="image" src="https://github.com/user-attachments/assets/b21509a8-a524-4034-bab4-7db748fca86f" />

<img width="334" height="153" alt="image" src="https://github.com/user-attachments/assets/3c8dc4a8-fd51-4526-a356-ea2411021293" />

<img width="444" height="178" alt="image" src="https://github.com/user-attachments/assets/fc539f91-a9e6-46e5-86a2-fece94c3a2a7" />

<img width="325" height="222" alt="image" src="https://github.com/user-attachments/assets/8d6ab8ea-9629-4e50-bc99-57a62401b401" />

Edit `ubuntu-lv` to use the full 48 G

<img width="304" height="217" alt="image" src="https://github.com/user-attachments/assets/909fea80-04e4-41d9-9448-62ab29c650f4" />

Created username and password

<img width="455" height="109" alt="image" src="https://github.com/user-attachments/assets/cd7bac75-53f2-40aa-ae97-a17689008a7c" />

<img width="380" height="111" alt="image" src="https://github.com/user-attachments/assets/d50f9ccf-a5f8-4869-b3f3-8bd095e3b2c8" />

<img width="460" height="154" alt="image" src="https://github.com/user-attachments/assets/3a3ed1ad-af48-48ff-8227-019439d1d632" />

Remove the media then reboot

<img width="580" height="298" alt="image" src="https://github.com/user-attachments/assets/eb6bc3c9-bf49-4165-bc0d-d63061dbfcae" />

Now I am logged in and going to check connectivity, I will run: `ping -c 3 10.0.20.1`, `ping -c 3 google.com`

Both were successful 

<img width="307" height="136" alt="image" src="https://github.com/user-attachments/assets/a8b817e4-e666-49fe-93a7-18ce8be8db0b" />

I run `wget -O splunk-10.4.3-4174a2deda5d-linux-amd64.deb "https://download.splunk.com/products/splunk/releases/10.4.3/linux/splunk-10.4.3-4174a2deda5d-linux-amd64.deb"`

<img width="529" height="88" alt="image" src="https://github.com/user-attachments/assets/17f90cb0-b6c6-421b-9c66-b0e5f76f415c" />

The install failed silently (indicted by the empty `/opt`) , so I run `cd ~` then `sudo dpkg -i "splunk-10.4.3-4174a2deda5d-linux-amd64.deb"`

Now that its correctly installed, I run the first setup `sudo /opt/splunk/bin/splunk start --accept-licens`

I cannot boot into web ui, and its because I never added a firewall rule to allow over port 8000

<img width="493" height="437" alt="image" src="https://github.com/user-attachments/assets/32942851-7b23-445b-9fd3-001fb09d4509" />

Now I am in the web ui.

First thing I do is verify the license and hostname, within `Settings` -> `Server Settings`

<img width="520" height="244" alt="image" src="https://github.com/user-attachments/assets/35d6cd4c-fcb2-46da-9329-d07d30e67f48" />

<img width="472" height="58" alt="image" src="https://github.com/user-attachments/assets/46d2cced-9b8d-4ab2-ba6b-95b92a0c19cc" />

<img width="481" height="127" alt="image" src="https://github.com/user-attachments/assets/b0117233-44f7-4278-a09b-ab1daa9997c0" />

Then make sure Splunk is persistent 

I realized I installed splunk into my home directory instead of root, so I switched that over.

Then finished with a snapshot.

<img width="496" height="137" alt="image" src="https://github.com/user-attachments/assets/c6044ad0-5a7c-4eb1-8e03-f48b481e7a69" />

# Cassandra + Elasticsearch VM

Same steps as before, creating a new VM.

<img width="363" height="105" alt="image" src="https://github.com/user-attachments/assets/398aeb7e-c3c0-4dc3-9b3e-8e499bd54a43" />

<img width="354" height="130" alt="image" src="https://github.com/user-attachments/assets/22766483-f711-40ec-b8a1-0d83726bd052" />

<img width="361" height="112" alt="image" src="https://github.com/user-attachments/assets/454662b4-fe06-4cbb-b62b-b7d4f30c2f75" />

<img width="361" height="140" alt="image" src="https://github.com/user-attachments/assets/47239d68-c0ce-4a40-870d-06dc4c519d4a" />

<img width="361" height="81" alt="image" src="https://github.com/user-attachments/assets/0a4d510d-7544-4dbf-810c-a9392e0c2bce" />

<img width="361" height="69" alt="image" src="https://github.com/user-attachments/assets/bbb339b7-f702-44c0-969f-9f03243449e5" />

<img width="364" height="109" alt="image" src="https://github.com/user-attachments/assets/0e31c200-58ff-42c9-89b1-73873de1f09f" />

# Ubuntu Setup for The Hive & Cortex

<img width="433" height="148" alt="image" src="https://github.com/user-attachments/assets/7d887d02-8c3b-4e5a-ba05-e1115c341266" />

<img width="247" height="155" alt="image" src="https://github.com/user-attachments/assets/02baec55-7d94-43dc-b2ac-4a5d088941eb" />

<img width="473" height="89" alt="image" src="https://github.com/user-attachments/assets/12b0dfa8-ce56-40e4-8b23-b20631dfbfbe" />

<img width="454" height="136" alt="image" src="https://github.com/user-attachments/assets/1b263731-9500-43bf-b7b1-5072ae884bdf" />

<img width="454" height="205" alt="image" src="https://github.com/user-attachments/assets/57cb15a1-4ab9-4a8f-97fc-f91d0db6e488" />

Same change to the storage, edit it to use the full `58G` capacity.

<img width="295" height="231" alt="Screenshot 2026-09-27 121531" src="https://github.com/user-attachments/assets/d07a34c3-35e6-4015-af6b-97083bdcbfb6" />

<img width="310" height="211" alt="image" src="https://github.com/user-attachments/assets/a06f3bac-65ed-4c7f-a05a-1be69ee05366" />

<img width="434" height="120" alt="image" src="https://github.com/user-attachments/assets/5514ca67-abe6-42dd-a60c-083af25333b0" />

After finishing install, remove media to continue.

Quick connectivity check with `ping -c 3 10.0.20.1` and `ping -c 3 google.com`

<img width="319" height="133" alt="image" src="https://github.com/user-attachments/assets/53d719b6-7f66-49c8-b1c0-874566601d9c" />

The hive and cortex both require Java so I will first install that

`sudo apt update && sudo apt install -y openjdk-11-jdk`

Then I will download and store Cassandra's signing key

`sudo mkdir -p /usr/share/keyrings curl -1sLf 'https://downloads.apache.org/cassandra/KEYS' | sudo gpg --dearmor -o /usr/share/keyrings/cassandra-archive-keyring.gpg`

I added in the ability to copy and paste using 

Then to add into the allow list on the host machine `qm set {vmid} -vga qxl,clipboard=vnc`

Then `echo 'deb [signed-by=/usr/share/keyrings/cassandra-archive-keyring.gpg] https://debian.cassandra.apache.org 41x main' | sudo tee /etc/apt/sources.list.d/cassandra.sources.list` then `sudo apt update` then `sudo apt install -y cassandra` then verify with `sudo systemctl status cassandra`

<img width="437" height="39" alt="image" src="https://github.com/user-attachments/assets/0486d99d-86b4-4529-871f-634d64b47d34" />

Moving onto downloading elastic searches signing key with 

`curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg`

Then add Elasticsearch repository with `echo 'deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main' | sudo tee /etc/apt/sources.list.d/elastic-7.x.list` then `sudo apt update`

Then install Elasticsearch with `sudo apt install -y elasticsearch`

Then run it with `sudo systemctl enable elasticsearch` then `sudo systemctl start elasticsearch` then `sudo systemctl status elasticsearch`

Data VM is functionally installed — but by default, both Cassandra and Elasticsearch only listen on localhost, which won't work since TheHive/Cortex will live on a separate VM and need to reach them over the network.

sudo nano /etc/cassandra/cassandra.yaml Find listen_address: localhost and change it to listen_address: 10.0.20.12. Find rpc_address: localhost and change it to rpc_address: 10.0.20.12. Save (Ctrl+O, Enter) and exit (Ctrl+X).

<img width="111" height="46" alt="image" src="https://github.com/user-attachments/assets/927992e9-afc2-4a09-984d-fd8f8abb748a" />

<img width="340" height="40" alt="image" src="https://github.com/user-attachments/assets/4e8b55a2-94c0-473a-bcb2-fd0be75431fe" />

Then restart Cassandra with `sudo systemctl restart cassandra` then `sudo systemctl status cassandra` Confirm it comes back up active with the new bind address.

I had to make another edit in the file 

<img width="274" height="99" alt="image" src="https://github.com/user-attachments/assets/85e19dbb-b06d-489d-85fa-89ef68de0939" />

Confirmed it is listening correctly

<img width="445" height="333" alt="image" src="https://github.com/user-attachments/assets/6605322b-854c-43db-bb94-00069ea4e59a" />

Now Ill do the same for Elasticsearch

`sudo nano /etc/elasticsearch/elasticsearch.yml`

My yaml file was empty so I am going to confirm it exists with `ls -la /etc/elasticsearch/elasticsearch.yml`

<img width="312" height="40" alt="image" src="https://github.com/user-attachments/assets/eb6a45db-7c9c-44a7-a99e-8b5d604b09c6" />

I will remove the one I created with `sudo rm /etc/elasticsearch/elasticsearch.yml`

Confirmed it was there, and I was putting `yaml` instead of `yml`

Then go back to editing the yml file

<img width="297" height="51" alt="image" src="https://github.com/user-attachments/assets/f78cb0b6-94db-4721-9a8c-2d232f79ad78" />

<img width="372" height="30" alt="image" src="https://github.com/user-attachments/assets/d60b06de-a12c-48da-8be2-6611f1de3613" />

Restart elasticsearch `sudo systemctl restart elasticsearch`

Verify its listening `sudo ss -tlnp | grep 9200` Should show 10.0.20.12:9200 listening

<img width="348" height="28" alt="image" src="https://github.com/user-attachments/assets/c15a47cf-8508-40e0-a887-fa0e6c850c47" />

I confirm it responds with `curl http://10.0.20.12:9200`

Next I will make a snap shot

<img width="405" height="108" alt="image" src="https://github.com/user-attachments/assets/60d7fa6a-f239-4be3-9d80-4896c149e67e" />

Quick rundown before new VM creation

VM 100 — pfSense (firewall)
VM 101 — Wazuh (all-in-one)
VM 102 — Splunk
VM 103 — soc-thehive-data-01 (Cassandra + Elasticsearch) — just finished
VM 104 — soc-thehive-app-01 (TheHive + Cortex) — building now

# The Hive + Cortex VM

<img width="365" height="108" alt="image" src="https://github.com/user-attachments/assets/82b70564-c5a2-4cfb-b2c9-66d3c9a7225e" />

<img width="358" height="123" alt="image" src="https://github.com/user-attachments/assets/7814b97a-d6d2-4224-8f40-fa4ca5c7d137" />

<img width="357" height="120" alt="image" src="https://github.com/user-attachments/assets/c2c16eb4-1e4e-423c-b8a7-dbe57bcbf0a1" />

<img width="360" height="145" alt="image" src="https://github.com/user-attachments/assets/a57a23b2-fb90-447d-a766-38e2ad8d47a1" />

<img width="363" height="85" alt="image" src="https://github.com/user-attachments/assets/7c60f309-be77-4dcc-84f0-
7412ca02456b" />

<img width="372" height="72" alt="image" src="https://github.com/user-attachments/assets/ac0bcfa0-fce9-491b-8e3c-d4228d9fb964" />

<img width="367" height="117" alt="image" src="https://github.com/user-attachments/assets/2a49a98f-acd2-44f2-8e1b-fad7d5f72415" />

Same Ubuntu setup as before, I wont share here. And also confirmed connectivity

# Install Java and jq

`sudo apt install -y jq openjdk-11-jdk jq` -  lets us pull the current package URL programmatically instead of hardcoding a version number that'll go stale.

I ran into many errors, too many to list, so just know its installed now.

Next I move to installing the Hive `wget -O /tmp/thehive.deb "$DEB_URL"` then `sudo dpkg -i /tmp/thehive.deb`

Next is to configure TheHive to point at your Cassandra+Elasticsearch VM

1. `openssl rand -hex 32` copy the text
2. `sudo nano /etc/thehive/application.conf`
3. Edit with hive IP, removing any 127.0.0.0 IPs present
4. Then `sudo bash -c 'cat > /etc/thehive/secret.conf << _EOF_
play.http.secret.key="$(cat /dev/urandom | tr -dc "a-zA-Z0-9" | fold -w 64 | head -n 1)"
_EOF_'`

Then I start the hive `sudo systemctl start thehive` then `sudo systemctl status thehive` then `sudo tail -n 50 /var/log/thehive/application.log`

Confirmed it was up and running with `curl -sI http://localhost:9000 -X GET`

Next is to add a firewall rule in pfsense

<img width="496" height="237" alt="image" src="https://github.com/user-attachments/assets/cd1e346b-f077-420c-b965-ef69a4ba76bd" />

`Source = 192.168,0,0/24`

`Destination = 10.0.20.0/24`

`Port = 9000`

`Action = Pass`

I check the current states to see if `10.0.20.13` is on the list, which its not.

<img width="207" height="271" alt="image" src="https://github.com/user-attachments/assets/a2bc3d3f-c90a-41fe-8186-c589c18ec4d6" />

Then I check states from my IP to `10.0.20.13`, which I dont have either.

<img width="213" height="334" alt="image" src="https://github.com/user-attachments/assets/e0699849-3bf7-4893-8a4a-efe5962493b8" />

I then run a packet capture to troubleshoot

<img width="485" height="403" alt="image" src="https://github.com/user-attachments/assets/548095c7-a3ec-4a55-84c0-433588170d77" />

Try to connect to the web ui, the stop the capture, and no packets were captured.

I double check my IP address

<img width="345" height="106" alt="image" src="https://github.com/user-attachments/assets/a9781511-3334-44f3-a4a8-81b7ec506cfd" />

Then I will trace the route from my pc with `tracert -d 10.0.20.13`

<img width="261" height="199" alt="image" src="https://github.com/user-attachments/assets/5d6116f0-ba6c-4cdb-9a08-26b994595b6e" />

In my windows pc I `route print` and see my entry for `10.0.20.0` is no longer there 

I add it with  `route add 10.0.20.0 mask 255.255.255.0 192.168.0.254`

Re do the packet capture, and now I see the entries I am looking for

<img width="274" height="178" alt="image" src="https://github.com/user-attachments/assets/5555d2d1-e580-4543-9e86-8ea1e0ee0ef9" />

I double check the states are captured now

<img width="219" height="283" alt="image" src="https://github.com/user-attachments/assets/e7f03e65-75e1-4325-83d6-66df3b213a2a" />

On 103 I check to see if cassandra and elastic are listening with `sudo ss -tlnp | grep 9042` `sudo ss -tlnp | grep 9200`

<img width="345" height="28" alt="image" src="https://github.com/user-attachments/assets/c534496a-d056-4186-b9f0-cd4c28076ad0" />

Then check the hives status `sudo systemctl status thehive`

<img width="310" height="52" alt="image" src="https://github.com/user-attachments/assets/5ebf9b5d-c4af-425c-a823-b2559bef6a71" />

Then restart the hive `sudo systemctl restart thehiv`

<img width="297" height="103" alt="image" src="https://github.com/user-attachments/assets/cefecd23-83c4-4f42-b009-a81dc28de4be" />

Then see if elastic is even running with `systemctl status elasticsearch` then `systemctl is-active elasticsearch` then `ps aux | grep elasticsearch`

<img width="279" height="37" alt="image" src="https://github.com/user-attachments/assets/e2fec213-fbe3-41f0-b6ee-b0ba273478ec" />

Seems my shell was stuck, I restarted it.

Double checked its running.

<img width="118" height="133" alt="image" src="https://github.com/user-attachments/assets/cf67cc66-48b2-403e-9790-95576b9f190d" />

Then completed the initial admin set up, then took a snapshot of each VM.

<img width="258" height="92" alt="image" src="https://github.com/user-attachments/assets/950b82c4-d96d-402c-8b2b-c6cfb8dbf851" />

<img width="271" height="89" alt="Screenshot 2026-10-07 175725" src="https://github.com/user-attachments/assets/d96beb66-c26c-4d42-ac39-2d242d09de54" />

# Cortex Install

On 104 I run `curl -s https://cortex.download.strangebee.com/manifest.json -o /tmp/cortex-manifest.json`

Confirm with `ls -la /tmp/cortex-manifest.json`

Then confirm the version with `LATEST=$(cat /tmp/cortex-manifest.json | jq -r '.index.latest.cortex')`

Confirm the url `DEB_URL=$(cat /tmp/cortex-manifest.json | jq -r ".products.cortex.all_versions[\"$LATEST\"].deb.url")`

Then install with `wget -O /tmp/cortex.deb "$DEB_URL"` then `sudo dpkg -i /tmp/cortex.deb`

<img width="439" height="178" alt="image" src="https://github.com/user-attachments/assets/1a508fd9-b13d-423c-ada6-745f2cb58ad1" />

Next I will point Elastic to Cortex `nano /etc/cortex/application.conf`

Set the secret key and changed the local IP to `10.0.20.12`

Start Cortex `sudo systemctl start cortex`

I am getting errors 

<img width="530" height="109" alt="image" src="https://github.com/user-attachments/assets/c0a00d1b-8053-4187-a75a-347d04df3269" />

The fix: `sudo chown -R cortex:cortex /opt/cortex` then I could start it

I confirmed I can connect

<img width="303" height="125" alt="image" src="https://github.com/user-attachments/assets/24d9dfbb-9372-4d58-8524-e9e43e76f0eb" />

# Adding Firewall Rule for Cortex

`Add` new rule in pfsense

<img width="511" height="193" alt="image" src="https://github.com/user-attachments/assets/68e283f7-38f7-44d1-89af-2e372de62075" />

Action = `Pass`, Interface = `WAN`, Address Family = `IPv4`, Protocol = `TCP`, Source = `192.168.0.0/24`, Destination = `10.0.20.0/24`

<img width="468" height="31" alt="image" src="https://github.com/user-attachments/assets/673e7903-0fb4-4f1a-98c7-9e48c21c0270" />

Test Cortex by going to `http://10.0.20.13:9001`

# Cortex Setup

Create admin account

<img width="501" height="306" alt="image" src="https://github.com/user-attachments/assets/e8ed9c9f-63dd-46f4-9d59-2a9f79da69e1" />


I add an orginization, name it and add a description
`Lab` and `Lab Cortex`

<img width="490" height="150" alt="image" src="https://github.com/user-attachments/assets/7f0cf8a3-f84d-43f9-bf57-03c0ff952d95" />

Click on the lab, and add a user

Login: `Lab_Cortex_User01`
Name: `John Smith`
Roles: `read, analyze, orgadmin`

Create an API key for the user

<img width="654" height="117" alt="image" src="https://github.com/user-attachments/assets/d49febbd-9a09-428f-a3cd-750941add188" />

Logged out of the super admin then back into the newly created account

# Docker install

Next I need to hook up Docker, on 104 I run `sudo apt install -y docker.io`

Once done I run `sudo systemctl enable --now docker`

Then `sudo usermod -aG docker cortex` (lets the cortex user use Docker. -aG appends, so it won’t wipe its existing groups.)

And double check its running `sudo systemctl status docker`

<img width="288" height="67" alt="image" src="https://github.com/user-attachments/assets/03bfebef-b61a-49a9-9426-c171d8555976" />

I restart cortex `sudo systemctl restart cortex` then check with `sudo systemctl status cortex`

I will test a couple analyzers to test the whole chain.

<img width="939" height="125" alt="image" src="https://github.com/user-attachments/assets/50d182ff-5e6d-4932-934c-4dec3a8c18af" />

<img width="930" height="56" alt="image" src="https://github.com/user-attachments/assets/98f18061-c9ac-4a44-b581-94d9e8bdfb75" />

<img width="451" height="214" alt="image" src="https://github.com/user-attachments/assets/aa2a38a2-c95a-4eca-9646-f900b7b4758e" />

It was successful 

<img width="949" height="219" alt="image" src="https://github.com/user-attachments/assets/4612bed1-fe12-444c-baae-8c0bd906704e" />

Created a snapshot

<img width="429" height="202" alt="image" src="https://github.com/user-attachments/assets/92c9a81e-90ba-4678-b333-fa9345e52014" />

# The Hive to Cotrex Link

I go to `platform management` -> `connectors` -> `Cortex tab` and set up a new server

<img width="207" height="55" alt="image" src="https://github.com/user-attachments/assets/3186802d-e784-45e8-8832-f89e7143a3e7" />

Set up an orginization

<img width="194" height="232" alt="image" src="https://github.com/user-attachments/assets/44412138-e180-4684-8f5e-a55c7c23591f" />

Add my admin to the org

Add a new user to the org, sign out of admin, sign back into new account

Create a case

<img width="439" height="447" alt="image" src="https://github.com/user-attachments/assets/ca0f7671-d452-4346-850b-c1480537facc" />

<img width="349" height="340" alt="image" src="https://github.com/user-attachments/assets/13c475f6-9fa4-4c8b-9aa3-bad9245a94d3" />

Go to `observables` then create one

<img width="364" height="202" alt="image" src="https://github.com/user-attachments/assets/52b3f5ca-1277-4c92-a04c-22d85a5f31e4" />

Test and it worked!

<img width="229" height="180" alt="image" src="https://github.com/user-attachments/assets/b4c59ffe-b331-4f55-a5f1-5509d07ce354" />

# Wazuh to TheHive Connection

First I create a Wazuh user within the hive

<img width="415" height="292" alt="image" src="https://github.com/user-attachments/assets/6fb9ad6c-2f86-4a12-aeeb-4ac8a4cfc791" />

Create an API key for the user

Test ping from wazuh to hive and vise versa.

Test connection to TheHive API port with `curl -s -o /dev/null -w "%{http_code}\n" http://10.0.20.13:9000`

<img width="312" height="126" alt="image" src="https://github.com/user-attachments/assets/ce784f10-4adf-400f-887f-2ca11c260f98" />

200 means the Wazuh VM can reach TheHive’s API port. The folder also contains stock wrappers (slack, shuffle and others). Their bash launcher is generic: it runs the .py file with the same name from the same folder using Wazuh’s embedded Python. So we copy one and name it for our integration. Custom integration names have to start with custom-.


First I create the wrapper with `sudo cp /var/ossec/integrations/slack /var/ossec/integrations/custom-w2thive`

Then open a new file for the script `sudo nano /var/ossec/integrations/custom-w2thive.py

Then I paste:
`#!/var/ossec/framework/python/bin/python3
# custom-w2thive: forward Wazuh alerts to TheHive 5 (POST /api/v1/alert)
import hashlib
import json
import sys

import requests

LOG_FILE = "/var/ossec/logs/integrations.log"


def log(msg):
    with open(LOG_FILE, "a") as f:
        f.write("custom-w2thive: " + msg + "\n")


def thehive_severity(level):
    if level >= 12:
        return 4
    if level >= 10:
        return 3
    if level >= 7:
        return 2
    return 1


def main():
    alert_file, api_key, hook_url = sys.argv[1], sys.argv[2], sys.argv[3]
    with open(alert_file) as f:
        alert = json.load(f)

    rule = alert.get("rule", {})
    agent = alert.get("agent", {})
    data = alert.get("data", {})
    level = int(rule.get("level", 0))

    ref = str(alert.get("id") or hashlib.sha1(
        json.dumps(alert, sort_keys=True).encode()).hexdigest())
    title = "Wazuh {}: {} on {}".format(
        rule.get("id", "?"), rule.get("description", "alert"),
        agent.get("name", "unknown"))
    description = (
        "**Rule:** {} (level {})\n\n"
        "**Agent:** {} ({})\n\n"
        "**Location:** {}\n\n"
        "**Log:**\n```\n{}\n```"
    ).format(rule.get("id"), level, agent.get("name"),
             agent.get("ip", "n/a"), alert.get("location"),
             str(alert.get("full_log", ""))[:2000])

    tags = ["wazuh", "rule:" + str(rule.get("id"))]
    tags += ["group:" + g for g in rule.get("groups", [])][:5]

    observables = []
    if data.get("srcip"):
        observables.append(
            {"dataType": "ip", "data": data["srcip"], "tags": ["wazuh"]})

    payload = {
        "type": "wazuh_alert",
        "source": "wazuh",
        "sourceRef": ref,
        "title": title,
        "description": description,
        "severity": thehive_severity(level),
        "tlp": 2,
        "pap": 2,
        "tags": tags,
        "observables": observables,
    }

    try:
        r = requests.post(
            hook_url.rstrip("/") + "/api/v1/alert",
            headers={"Authorization": "Bearer " + api_key},
            json=payload, timeout=15)
        log("HTTP {} for rule {} ({})".format(r.status_code, rule.get("id"), ref))
        if r.status_code >= 300:
            log(r.text[:300])
    except Exception as e:
        log("error: {}".format(e))


if __name__ == "__main__":
    main()`

    
Then change the ownership with `sudo chown root:wazuh /var/ossec/integrations/custom-w2thive /var/ossec/integrations/custom-w2thive.py`

Then `sudo chmod 750 /var/ossec/integrations/custom-w2thive /var/ossec/integrations/custom-w2thive.py`

this checks that the paste didn’t mangle the code. No output means it’s fine. `sudo /var/ossec/framework/python/bin/python3 -m py_compile /var/ossec/integrations/custom-w2thive.py`

Next is for me to tell Wazuh when to fire it with `sudo nano /var/ossec/etc/ossec.conf`

Then paste in after </global>:
`  <integration>
    <name>custom-w2thive</name>
    <hook_url>http://10.0.20.13:9000</hook_url>
    <api_key>PASTE_YOUR_THEHIVE_API_KEY_HERE</api_key>
    <level>5</level>
    <alert_format>json</alert_format>
  </integration>`

<img width="372" height="277" alt="Screenshot 2026-10-10 111207" src="https://github.com/user-attachments/assets/2efb486c-cac0-4087-bcc5-f39164e43690" />

Restarted `wauh-manager`
  
<img width="448" height="54" alt="image" src="https://github.com/user-attachments/assets/331a132d-d01f-47ae-93a4-1ee0a803e5c1" />

I check the integration daemon is up then generate a test event.

<img width="373" height="127" alt="image" src="https://github.com/user-attachments/assets/9cc083b9-4067-424c-8f18-2e5cc57390a7" />

Then check if the script ran with `sudo tail -n 15 /var/ossec/logs/integrations.log`

<img width="322" height="85" alt="image" src="https://github.com/user-attachments/assets/18d89dd3-eac0-4256-81dd-a0944a3fc46c" />

It is successful. I then log in as the wazuh user and check alerts

<img width="955" height="456" alt="image" src="https://github.com/user-attachments/assets/024902f4-e54d-4cb9-8dfe-e827d07b2911" />

I am SSh'ed into my wazuh server, I will run a test without being sshed

<img width="290" height="171" alt="image" src="https://github.com/user-attachments/assets/3b65d4d1-4404-439d-a2b1-0eb24d71619d" />

<img width="100" height="154" alt="image" src="https://github.com/user-attachments/assets/3e714cf6-b1cc-4d65-befa-4d8337a1da67" />

Next I will raise the threshold of the script from 5 to 7 to decrease flood

<img width="72" height="19" alt="image" src="https://github.com/user-attachments/assets/99ed3be0-cfdc-46ff-b565-9b120833b048" />

<img width="70" height="19" alt="image" src="https://github.com/user-attachments/assets/9fce899f-d79b-40f0-800d-908e42cd6352" />

Then restart Wazuh, and snapshot 101 and 104 VM


# End of Phase 2

