---
description: https://tryhackme.com/room/ra
---

# Ra

## Nmap

```
nmap 10.10.110.170 -p- -sS -sV   

PORT      STATE SERVICE             VERSION
80/tcp    open  http                Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec        Microsoft Windows Kerberos (server time: 2022-04-11 16:48:05Z)
135/tcp   open  msrpc               Microsoft Windows RPC
139/tcp   open  netbios-ssn         Microsoft Windows netbios-ssn
389/tcp   open  ldap                Microsoft Windows Active Directory LDAP (Domain: windcorp.thm0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http          Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
2179/tcp  open  vmrdp?
3268/tcp  open  ldap                Microsoft Windows Active Directory LDAP (Domain: windcorp.thm0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server       Microsoft Terminal Services
5222/tcp  open  jabber              Ignite Realtime Openfire Jabber server 3.10.0 or later
5223/tcp  open  ssl/jabber          Ignite Realtime Openfire Jabber server 3.10.0 or later
5229/tcp  open  jaxflow?
5262/tcp  open  jabber              Ignite Realtime Openfire Jabber server 3.10.0 or later
5263/tcp  open  ssl/jabber
5269/tcp  open  xmpp                Wildfire XMPP Client
5270/tcp  open  ssl/xmpp            Wildfire XMPP Client
5275/tcp  open  jabber              Ignite Realtime Openfire Jabber server 3.10.0 or later
5276/tcp  open  ssl/jabber
5985/tcp  open  http                Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
7070/tcp  open  http                Jetty 9.4.18.v20190429
7443/tcp  open  ssl/http            Jetty 9.4.18.v20190429
7777/tcp  open  socks5              (No authentication; connection failed)
9090/tcp  open  zeus-admin?
9091/tcp  open  ssl/xmltec-xmlmail?
9389/tcp  open  mc-nmf              .NET Message Framing
49670/tcp open  msrpc               Microsoft Windows RPC
49674/tcp open  ncacn_http          Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc               Microsoft Windows RPC
49676/tcp open  msrpc               Microsoft Windows RPC
49695/tcp open  msrpc               Microsoft Windows RPC
```

We have a large number of ports open on the target system. Firstly we are going to take a look at port 80.

![](<../../.gitbook/assets/image (66).png>)

Part way down the page we find some employee information.

![](<../../.gitbook/assets/image (1013).png>)

Viewing the page source we see a list of usernames.

![](<../../.gitbook/assets/image (1600).png>)

From the source we also pick up the FQDN of fire.windcorp.thm.

Running the discovered usernames against `kerbrute` reveals some are valid users accounts.

```bash
kerbrute userenum 'user.txt' --dc <IP> --domain windcorp.thm -v
```

![](<../../.gitbook/assets/image (562).png>)

![](<../../.gitbook/assets/image (364).png>)

A further look through the main web page shows an opportunity to reset user passwords based on a security questions.

As we know Lily's username from the page source, we can also answer her secret questions as the page source for her profile image is named "LilyleandSparky.jpg".

![](<../../.gitbook/assets/image (25) (2).png>)

![](<../../.gitbook/assets/image (1119).png>)

After entering the relevant information we are given a new password for the user _lilyle_.

![](<../../.gitbook/assets/image (252).png>)

Which, when testing with `crackmapexec` confirms the credentials are valid.

![](<../../.gitbook/assets/image (13) (2).png>)

Using smbmap with lily's account we some readable shares.

```
smbmap -H '10.10.110.170' -u 'lilyle' -p <Password> -q
```

![](<../../.gitbook/assets/image (1023).png>)

As well as the first flag for the room.

```
smbmap -H '10.10.110.170' -u 'lilyle' -p <Password> -R --depth 30
```

![](<../../.gitbook/assets/image (1069).png>)
