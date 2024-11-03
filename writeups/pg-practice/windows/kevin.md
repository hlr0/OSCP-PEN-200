---
description: PG Practice Kevin writeup
---

# Kevin

## Nmap

```
sudo nmap   192.168.214.45 -p- -sS -sV

PORT      STATE SERVICE            VERSION
80/tcp    open  http               GoAhead WebServer
135/tcp   open  msrpc              Microsoft Windows RPC
139/tcp   open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds       Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ssl/ms-wbt-server?
3573/tcp  open  tag-ups-1?
49152/tcp open  msrpc              Microsoft Windows RPC
49153/tcp open  msrpc              Microsoft Windows RPC
49154/tcp open  msrpc              Microsoft Windows RPC
49155/tcp open  msrpc              Microsoft Windows RPC
49158/tcp open  msrpc              Microsoft Windows RPC
49160/tcp open  msrpc              Microsoft Windows RPC
Service Info: Host: KEVIN; OS: Windows; CPE: cpe:/o:microsoft:windows
```

Default page for Port 80 at: [http://192.168.214.45/index.asp](http://192.168.214.45/index.asp) Takes us to a login screen for HP Power Manager. A quick Google search reveals the default credentials are `admin:admin`.

![](<../../../.gitbook/assets/image (1005) (1).png>)

After logging in moving over to the help tab reveals version information.

![](<../../../.gitbook/assets/image (1006).png>)

`Searchsploit` reveals HP Power Manager is vulnerable to a remote buffer overflow given **CVE-2009-3999.**

**Description:**

Stack-based buffer overflow in goform/formExportDataLogs in HP Power Manager before 4.2.10 allows remote attackers to execute arbitrary code via a long file Name parameter.

![](<../../../.gitbook/assets/image (1007).png>)

{% embed url="https://www.exploit-db.com/exploits/18015" %}

The following MSF module was used: `exploit/windows/http/hp_power_manager_filename`.

![](<../../../.gitbook/assets/image (1008) (1).png>)
