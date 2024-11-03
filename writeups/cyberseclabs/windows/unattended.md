---
description: https://www.cyberseclabs.co.uk/labs/info/Unattended/
---

# Unattended

![](<../../../.gitbook/assets/image (514).png>)

## Scanning and Enumeration

### Nmap

```
nmap 172.31.1.24 -p- -A

PORT      STATE SERVICE       VERSION
80/tcp    open  http          HttpFileServer httpd 2.3
|_http-server-header: HFS 2.3
|_http-title: HFS /
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: UNATTENDED
|   NetBIOS_Domain_Name: UNATTENDED
|   NetBIOS_Computer_Name: UNATTENDED
|   DNS_Domain_Name: Unattended
|   DNS_Computer_Name: Unattended
|   Product_Version: 10.0.17763
|_  System_Time: 2020-12-11T17:51:41+00:00
| ssl-cert: Subject: commonName=Unattended
| Not valid before: 2020-12-10T17:47:09
|_Not valid after:  2021-06-11T17:47:09
|_ssl-date: 2020-12-11T17:51:45+00:00; -1s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49672/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Port 445 (SMB)

I started off with checking SMB for null authentication as I usually do for any quick wins. These however did not return any decent results.

![](<../../../.gitbook/assets/image (515) (1).png>)

### Port 80 (HTTP)

Apart from Port 80 the remaining ports are used for remote management or RDP services so port 80 is likely the next best vector.

The root page of port 80 redirects us to:

![http://172.31.1.24/](<../../../.gitbook/assets/image (516) (1).png>)

## Exploitation

### Low Privilege Shell

Fortunately for myself I have dealt with a machine before using this which is Rejetto HttpFileServer 2.3 as shown in the bottom left. I am aware at this point of `metasploit` having a module for this so I will be proceeding with metasploit for this machine.

First off fire off metasploit and search for 'Rejetto'.

![](<../../../.gitbook/assets/image (517) (1).png>)

Select the module and set the appropriate options.

![](<../../../.gitbook/assets/image (519).png>)

Once the the correct options have been set run the exploit and you should get a shell as the user 'pink'.

![](<../../../.gitbook/assets/image (520).png>)

### Privilege Escalation

The name of the machine is a dead giveaway for privilege escalation. Unattend.xml which is a file used by unattended installs. If not properly sanitized can leave administrative credentials in plain text.

Metasploit has a module for checking against these `post/windows/gather/enum_unattend` set this module and set the correct session ID.

![](<../../../.gitbook/assets/image (521) (1).png>)

We now have obtained the following credentials: Administrator:cnt4weRAbtXMTSVV

I will use Impacket's psexec.py to connect as 'NT AUTHORITY\System'.

```
sudo python2 psexec.py /administrator:cnt4weRAbtXMTSVV@172.31.1.24 
```

![](<../../../.gitbook/assets/image (522).png>)

I was able to also connect with these credentials over RDP and WinRM.

![](<../../../.gitbook/assets/image (523).png>)
