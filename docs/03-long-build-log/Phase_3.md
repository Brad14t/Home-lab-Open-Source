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

<img width="120" height="145" alt="image" src="https://github.com/user-attachments/assets/4e03a632-18d2-4ca1-baf0-9af099e3955a" />

Create the top-level OU: `New-ADOrganizationalUnit -Name "LAB" -Path "DC=lab,DC=local"`

Create the second-level OUs: `"Admins","Service Accounts","Groups","Servers","Workstations","Users" | ForEach-Object { New-ADOrganizationalUnit -Name $_ -Path "OU=LAB,DC=lab,DC=local" }`

Create the department OUs: `"IT","HR","Finance","Sales" | ForEach-Object { New-ADOrganizationalUnit -Name $_ -Path "OU=Users,OU=LAB,DC=lab,DC=local" }`

Send newly joined computers to the Workstations OU instead of the default Computers container, so GPOs apply to them right away: `redircmp "OU=Workstations,OU=LAB,DC=lab,DC=local"`

<img width="453" height="203" alt="image" src="https://github.com/user-attachments/assets/d8270b0f-9440-4792-b7a3-fdef6ab5d89a" />

Rename my mistake "Financ" and remove accidental deletion: `Set-ADOrganizationalUnit -Identity "OU=Financ,OU=Users,OU=LAB,DC=lab,DC=local" -ProtectedFromAccidentalDeletion $false`

`Rename-ADObject -Identity "OU=Financ,OU=Users,OU=LAB,DC=lab,DC=local" -NewName "Finance"`

Then turn it back on `Set-ADOrganizationalUnit -Identity "OU=Finance,OU=Users,OU=LAB,DC=lab,DC=local" -ProtectedFromAccidentalDeletion $true`

Confirm: `Get-ADOrganizationalUnit -Filter 'Name -like "Fin*"' -Properties ProtectedFromAccidentalDeletion | Select-Object DistinguishedName,ProtectedFromAccidentalDeletion`

Next part is the groups, users, and services

Why: This gives realistic identities for GPO targeting, logons for Wazuh to monitor, and two deliberately weak service accounts as targets for Phase 6 attack/detect scenarios.

Groups: `"IT-Staff","HR-Staff","Finance-Staff","Sales-Staff","Workstation-Admins" | ForEach-Object { New-ADGroup -Name $_ -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=LAB,DC=lab,DC=local" }`

Standard users (2 per department): `$pw = Read-Host -AsSecureString "Initial user password"`

Define the users: `$users = @(@{S="jcarter";G="James";L="Carter";D="IT"},@{S="mlopez";G="Maria";L="Lopez";D="IT"},@{S="spatel";G="Sanjay";L="Patel";D="HR"},@{S="kwong";G="Karen";L="Wong";D="HR"},@{S="dkim";G="Daniel";L="Kim";D="Finance"},@{S="rjohnson";G="Rachel";L="Johnson";D="Finance"},@{S="tnguyen";G="Tom";L="Nguyen";D="Sales"},@{S="abrooks";G="Amy";L="Brooks";D="Sales"})`

Create each user and add them to their department group: `$users | ForEach-Object { New-ADUser -Name "$($_.G) $($_.L)" -GivenName $_.G -Surname $_.L -SamAccountName $_.S -UserPrincipalName "$($_.S)@lab.local" -Department $_.D -Path "OU=$($_.D),OU=Users,OU=LAB,DC=lab,DC=local" -AccountPassword $pw -Enabled $true; Add-ADGroupMember -Identity "$($_.D)-Staff" -Members $_.S }`

create an alternate admin account 

Why: Stop using the built-in Administrator for daily work. A separate named admin account leaves a clear audit trail, and the writeup can call this tiered admin practice.

`$adminPw = Read-Host -AsSecureString "adm-brad password"`

`New-ADUser -Name "adm-brad" -SamAccountName "adm-brad" -UserPrincipalName "adm-brad@lab.local" -Path "OU=Admins,OU=LAB,DC=lab,DC=local" -AccountPassword $adminPw -Enabled $true -Description "Personal domain admin account"`

`Add-ADGroupMember -Identity "Domain Admins" -Members "adm-brad"`

Next create the vulnerable accounts

`$svcPw = Read-Host -AsSecureString "Weak service account password"`

`New-ADUser -Name "svc-sql" -SamAccountName "svc-sql" -UserPrincipalName "svc-sql@lab.local" -Path "OU=Service Accounts,OU=LAB,DC=lab,DC=local" -AccountPassword $svcPw -Enabled $true -PasswordNeverExpires $true -Description "INTENTIONALLY VULNERABLE - Kerberoastable (lab)"`

`Set-ADUser -Identity "svc-sql" -ServicePrincipalNames @{Add="MSSQLSvc/sql01.lab.local:1433"}`

`New-ADUser -Name "svc-backup" -SamAccountName "svc-backup" -UserPrincipalName "svc-backup@lab.local" -Path "OU=Service Accounts,OU=LAB,DC=lab,DC=local" -AccountPassword $svcPw -Enabled $true -PasswordNeverExpires $true -Description "INTENTIONALLY VULNERABLE - AS-REP roastable (lab)"`

`Set-ADAccountControl -Identity "svc-backup" -DoesNotRequirePreAuth $true`

In production, service accounts would be gMSAs (auto-rotated 120-character passwords). These two are deliberately misconfigured, and the repo should say so clearly.

Confirm the accoutns

<img width="487" height="141" alt="image" src="https://github.com/user-attachments/assets/a4579c75-2a11-4aad-acd0-68046572d0d9" />

# Building DC02

Why: A second DC gives AD redundancy. It also gives you replication to monitor, and replication events are what DCSync detection looks for in Phase 6.

<img width="360" height="103" alt="image" src="https://github.com/user-attachments/assets/6465a58b-99d3-46c5-98b4-42deb7ca5445" />

<img width="354" height="142" alt="image" src="https://github.com/user-attachments/assets/63479007-90bd-4775-a177-7fb5cbea0017" />

<img width="358" height="156" alt="image" src="https://github.com/user-attachments/assets/a387b8df-5d16-426f-aa56-036c5e240a0d" />

<img width="360" height="154" alt="image" src="https://github.com/user-attachments/assets/4584edcd-b56a-4c46-b72a-19457e1f4e9f" />

<img width="358" height="78" alt="image" src="https://github.com/user-attachments/assets/8b1f025c-83b5-4f19-a073-b3bea62fbb47" />

<img width="360" height="104" alt="image" src="https://github.com/user-attachments/assets/366ed7ce-397d-47e1-8046-8aa2fa6ed066" />

<img width="291" height="230" alt="image" src="https://github.com/user-attachments/assets/8ae8ee72-f0b6-4aaf-969f-a17b034771d6" />

<img width="297" height="238" alt="image" src="https://github.com/user-attachments/assets/679eafec-de97-4751-aa85-b0eedd67c3e0" />

<img width="307" height="245" alt="image" src="https://github.com/user-attachments/assets/dce2fd05-845e-4ba0-9bee-f0fafaf3c62b" />

<img width="209" height="157" alt="image" src="https://github.com/user-attachments/assets/5f44287e-50d3-4177-a5cb-fa77df04805d" />

Attach the two with
DC02 `Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.30.10,127.0.0.1` and `Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.30.11,127.0.0.1` on DC01


Add `10.0.30.11` as secondary DNS

<img width="156" height="124" alt="image" src="https://github.com/user-attachments/assets/0e68f465-e64b-423a-b247-00a07edecd5f" />

Verify with `Get-ADDomainController -Filter * | Select-Object Name,IPv4Address,IsGlobalCatalog`

<img width="450" height="73" alt="image" src="https://github.com/user-attachments/assets/08e5055f-60cb-4e44-9c8d-af36ebe156e6" />

Then in proxmox shell take a snapshot of both
`qm snapshot 110 phase3-dc02-replicating --vmstate 1 --description "Two-DC forest, replication healthy"`
`qm snapshot 111 phase3-dc02-replicating --vmstate 1 --description "Two-DC forest, replication healthy"`


# Group Policy

Password + lockout policy 

On DC01 `gpmc.msc`

Forest → Domains → lab.local → right-click Default Domain Policy → Edit → Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies

<img width="324" height="190" alt="image" src="https://github.com/user-attachments/assets/719e098f-8bad-44bc-952a-6deddd015a28" />

<img width="391" height="161" alt="image" src="https://github.com/user-attachments/assets/6d561e56-c8c4-438e-81fb-9975ecc1fb88" />

<img width="385" height="100" alt="image" src="https://github.com/user-attachments/assets/de1600a5-7cf8-4e9f-9806-cbb100f0dc41" />

<img width="306" height="115" alt="image" src="https://github.com/user-attachments/assets/d892c394-1050-4ae7-8fbf-db87217027a1" />

<img width="313" height="90" alt="image" src="https://github.com/user-attachments/assets/fbb58d80-1376-47f1-87bf-5f17066dd521" />

Lockouts generate event 4740

# Audit Policy

`New-GPO -Name "LAB - Audit Policy" | New-GPLink -Target "DC=lab,DC=local"`

`Set-GPRegistryValue -Name "LAB - Audit Policy" -Key "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -ValueName "ProcessCreationIncludeCmdLine_Enabled" -Type DWord -Value 1`

Raise the Security log to 512 MB so events aren’t overwritten before Wazuh collects them:

`Set-GPRegistryValue -Name "LAB - Audit Policy" -Key "HKLM\Software\Policies\Microsoft\Windows\EventLog\Security" -ValueName "MaxSize" -Type DWord -Value 524288`

Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options

<img width="289" height="150" alt="image" src="https://github.com/user-attachments/assets/9340816c-7a9d-41d3-a38e-6fbd3626e810" />

Security Settings → Advanced Audit Policy Configuration → Audit Policies

<img width="376" height="355" alt="image" src="https://github.com/user-attachments/assets/0bd2510f-0950-4a53-bf7a-02857017edf8" />

# PowerShell logging (PowerShell only)

Why: Script block logging (event 4104) records the de-obfuscated code that actually runs, which is one of the best sources for catching malicious PowerShell.

`New-GPO -Name "LAB - PowerShell Logging" | New-GPLink -Target "DC=lab,DC=local"`

Script block logging:
`Set-GPRegistryValue -Name "LAB - PowerShell Logging" -Key "HKLM\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -ValueName "EnableScriptBlockLogging" -Type DWord -Value 1`

Module logging:
`Set-GPRegistryValue -Name "LAB - PowerShell Logging" -Key "HKLM\Software\Policies\Microsoft\Windows\PowerShell\ModuleLogging" -ValueName "EnableModuleLogging" -Type DWord -Value 1`

Log all modules:
`Set-GPRegistryValue -Name "LAB - PowerShell Logging" -Key "HKLM\Software\Policies\Microsoft\Windows\PowerShell\ModuleLogging\ModuleNames" -ValueName "*" -Type String -Value "*"`

These events go to the Microsoft-Windows-PowerShell/Operational channel

Windows Firewall for workstations (PowerShell only)

`New-GPO -Name "LAB - Windows Firewall" | New-GPLink -Target "OU=Workstations,OU=LAB,DC=lab,DC=local"`

Turn the firewall on for all profiles, block inbound, allow outbound, and log dropped packets:

`Set-NetFirewallProfile -PolicyStore "lab.local\LAB - Windows Firewall" -Profile Domain,Private,Public -Enabled True -DefaultInboundAction Block -DefaultOutboundAction Allow -LogBlocked True -LogMaxSizeKilobytes 16384`

Apply 
`gpupdate /force`

Confirm 
`Get-ADDefaultDomainPasswordPolicy`

<img width="279" height="135" alt="image" src="https://github.com/user-attachments/assets/72ecd829-d49c-4b0f-ad75-f3474664e6bc" />


Create backup of GPO's

`New-Item -ItemType Directory -Path C:\GPOBackups -Force; Backup-GPO -All -Path C:\GPOBackups`

Then take snapshots on both domain controlers

# Create Windows VM

<img width="361" height="99" alt="image" src="https://github.com/user-attachments/assets/82a961df-01b5-405d-b8aa-55bd086486ff" />

<img width="358" height="131" alt="image" src="https://github.com/user-attachments/assets/440cee54-ed6f-4e02-9afa-5b8a45093cd0" />

<img width="356" height="172" alt="image" src="https://github.com/user-attachments/assets/66f85a6c-a841-4ef0-9f62-b2aed4ea1ccf" />

<img width="357" height="130" alt="image" src="https://github.com/user-attachments/assets/8eee877a-14cd-4f1c-8b7d-d0e6636faedd" />

<img width="358" height="81" alt="image" src="https://github.com/user-attachments/assets/f834018e-629d-4643-9255-31085b16a22d" />

<img width="258" height="65" alt="image" src="https://github.com/user-attachments/assets/f742c349-6337-47dc-9b3e-91456c1a73bd" />

<img width="359" height="112" alt="image" src="https://github.com/user-attachments/assets/e7481f21-92f9-4a84-ba6e-c41c14c1beb9" />

<img width="296" height="154" alt="image" src="https://github.com/user-attachments/assets/fb2ff350-d918-4ff9-8be4-f1e931db2a0d" />

<img width="295" height="133" alt="image" src="https://github.com/user-attachments/assets/6665a6eb-78c2-4456-88f8-5e8e00be1db5" />

<img width="292" height="142" alt="image" src="https://github.com/user-attachments/assets/aeefe9d3-f4d0-4f8d-ab9c-d45eccec48c3" />

Create a local account

then run `virtio-win-guest-tools.exe`

Now check the DNS is correct with `10.0.30.10` and `10.0.30.11`

<img width="173" height="25" alt="image" src="https://github.com/user-attachments/assets/f2602933-4d80-4ada-b563-0bff6e4dcc3e" />

`Add-Computer -DomainName "lab.local" -NewName "WS01" -Credential "LAB\adm-brad" -Restart`

Confirm on DC01 the vm was added `Get-ADComputer -Filter 'Name -like "WS*" -or Name -like "DESKTOP*"' | Select-Object Name,DistinguishedName`

<img width="495" height="55" alt="image" src="https://github.com/user-attachments/assets/34a312de-49df-4720-877a-5d1010201208" />

Then log into standard user account created earlier `jcarter`

And check connection to the domain, then snapshot.

# Create Another Windows VM

Same steps as above 

name `corp-ws-02`, hostname `WS02`

<img width="361" height="100" alt="image" src="https://github.com/user-attachments/assets/9f6bfa81-de96-4ba8-af12-1419d8da886b" />

<img width="355" height="147" alt="image" src="https://github.com/user-attachments/assets/0634fe4a-31fd-4300-a5c6-653a30071e15" />

<img width="361" height="163" alt="image" src="https://github.com/user-attachments/assets/362537af-e888-4c24-869b-3c1847f0cc59" />

<img width="355" height="76" alt="image" src="https://github.com/user-attachments/assets/c6dff63a-864f-4932-979a-f2b3729b6330" />

<img width="271" height="79" alt="image" src="https://github.com/user-attachments/assets/c0801b28-2fd7-43c5-8e48-0f65507e6ce1" />

<img width="360" height="114" alt="image" src="https://github.com/user-attachments/assets/3d9e2a9e-f4bb-4748-b1aa-922eb4bbf87b" />

<img width="200" height="163" alt="image" src="https://github.com/user-attachments/assets/c2843ee4-26fa-4d57-8346-c6b0e2c4af00" />

<img width="175" height="28" alt="image" src="https://github.com/user-attachments/assets/dda50b81-93bb-4eb2-8636-0e7e7908e87a" />

<img width="358" height="70" alt="image" src="https://github.com/user-attachments/assets/a82e935f-8295-4c24-af25-3849a3346227" />

<img width="412" height="75" alt="image" src="https://github.com/user-attachments/assets/a52179e4-d08e-4d77-b0ec-83759b045e8b" />

All four Windows machines are built. The last step of Phase 3 is getting their logs into Wazuh.

# Sysmon + Wazuh agents

Why: Native Windows logging misses a lot. Sysmon adds detailed process, network, and file telemetry that the attack detections in Phase 6 depend on. The Wazuh agents ship all of it, plus Security and PowerShell logs, to the SIEM.

Approach
- Download everything once to DC01’s NETLOGON share. Every domain computer can read it, and it replicates to DC02.
- Install from that share on each machine.
- A Wazuh agent group named windows pushes extra log channels to all four agents at once.

on 101 Create the windows agent group (on the Wazuh manager)

`sudo /var/ossec/bin/agent_groups -a -g windows -q`

`sudo nano /var/ossec/etc/shared/windows/agent.conf`

Replace the contents with
`<agent_config os="Windows">
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
  <localfile>
    <location>Microsoft-Windows-PowerShell/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</agent_config>`

Verify syntax with `sudo /var/ossec/bin/verify-agent-conf`

Then Stage the installers on NETLOGON (DC01, PowerShell Admin)

`$d = "C:\Windows\SYSVOL\sysvol\lab.local\scripts\Deploy"`

`New-Item -ItemType Directory -Path $d -Force`

`Invoke-WebRequest https://download.sysinternals.com/files/Sysmon.zip -OutFile "$d\Sysmon.zip"`

Sysmon config. I’m using Olaf Hartong’s sysmon-modular, whose rule names map to MITRE ATT&CK techniques, which is useful for detection engineering. SwiftOnSecurity’s sysmonconfig-export.xml is the other common baseline.

`Invoke-WebRequest https://github.com/olafhartong/sysmon-modular/releases/latest/download/sysmonconfig.xml -OutFile "$d\Sysmon\sysmonconfig.xml"`

Repeat on DC01, DC02, WS01, and WS02. On the DCs, use PowerShell (Admin) as adm-brad. On the workstations, use Terminal (Admin) and elevate with adm-brad.

`& "\\lab.local\NETLOGON\Deploy\Sysmon\Sysmon64.exe" -accepteula -i "\\lab.local\NETLOGON\Deploy\Sysmon\sysmonconfig.xml"`

Install the Wazuh agent. -Wait is needed because msiexec otherwise returns before the install finishes. The agent name defaults to the hostname.

<img width="214" height="55" alt="image" src="https://github.com/user-attachments/assets/c9497f5c-8ec7-4aa7-b4ce-0892a016c82c" />

Confirmed all 4 have `Sysmon64` and `Wazuh` running

On Wazuh Manager (101) I check they are all there with `sudo /var/ossec/bin/agent_control -l`

<img width="285" height="80" alt="image" src="https://github.com/user-attachments/assets/f603700e-be79-47d5-89d5-86bd40325df8" />

Snapshot and this is the complete of phase 3













