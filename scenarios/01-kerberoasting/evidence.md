## Kerberoasting 

### Running the attack (from Kali)

I used **Impacket's GetUserSPNs**, a standard tool that automates the request.

My attacker box can't use the domain's DNS, so first I added a hosts entry so the
tool can reach the domain controller by name:

```bash
echo "10.0.30.10  dc01.lab.local lab.local" | sudo tee -a /etc/hosts
```

<details>
<summary>Command explained:</summary>

| Command Component | What it does |
| :--- | :--- |
| `echo` | **What it does:** Prints the text inside the quotes to the standard output. |
| `"10.0.30.10  dc01.lab.local lab.local"` | **The content:** It follows the standard format for a host file entry: [IP Address] [Canonical Hostname] [Aliases] |
| `sudo tee -a /etc/hosts` | The name of the specific service/container defined in your `docker-compose.yml`. |

</details>


Then I requested the service ticket, authenticating as a normal, non-admin domain
user — that's all Kerberoasting needs:

```bash
impacket-GetUserSPNs lab.local/<user>:'<password>' -dc-ip 10.0.30.10 -request -outputfile kerberoast.hashes
```
<details>
<summary>Command explained:</summary>

| Command Component | What it does |
| :--- | :--- |
| `impacket-GetUserSPNs` | **What it does:** A tool from the Impacket framework specifically designed to find and target accounts that have Service Principal Names (SPNs) registered in Active Directory. |
| `lab.local/<user>:'<password>'` | **What it does:**  Authenticates to the domain using valid low-privileged domain credentials. |
| `-dc-ip 10.0.30.10` | **What it does:** Specifies the IP address of the Domain Controller (DC) to talk to. In this case, it targets 10.0.30.10 (the host added to your system's host file in the previous step). |
| `-request` | **What it does:** Instructs the tool to explicitly request the TGS tickets from the Key Distribution Center (KDC) on the DC for any SPN-enabled accounts it finds. Without this flag, the tool would only list the accounts without downloading their hashes. |
| `-outputfile kerberoast.hashes` | **What it does:** Saves the retrieved Kerberos tickets into a local file named kerberoast.hashes. The tool automatically formats these hashes so they are directly compatible with password-cracking tools like Hashcat or John the Ripper. |

</details>


