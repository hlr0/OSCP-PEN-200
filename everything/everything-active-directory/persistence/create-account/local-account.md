---
description: https://attack.mitre.org/techniques/T1136/001/
---

# Local Account

**ATT\&CK ID:** [T1136.001](https://attack.mitre.org/techniques/T1136/001/)

**Permissions Required:** <mark style="color:red;">**Administrator**</mark> | <mark style="color:red;">**SYSTEM**</mark>

**Description**

Adversaries may create a local account to maintain access to victim systems. Local accounts are those configured by an organization for use by users, remote support, services, or for administration on a single system or service. With a sufficient level of access, the `net user /add` command can be used to create a local account. On macOS systems the `dscl -create` command can be used to create a local account.

Such accounts may be used to establish secondary credentialed access that do not require persistent remote access tools to be deployed on the system.

## **Techniques**

### **CMD**

```bash
# Create new user
net user /add "<Username>" "<Password>"

# Add to RDP Group
net localgroup "Remote Desktop Users" "<Username>" /add

# Add to local administrators
net localgroup administrators "<Username>" /add
```

### **PowerShell**

```bash
# Create new user 
New-LocalUser -Name "<Username>" -NoPassword

# Add to RDP Group
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "<Username>"

# Add to local administrators
Add-LocalGroupMember -Group "Administrators" -Member "<Username>"
```

## **Mitigation**

Limit the usage of local administrator accounts to be used for day-to-day operations that may expose them to potential adversaries.

## **Further Reading**

**PowerShell:** [https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/new-localuser?view=powershell-5.1](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/new-localuser?view=powershell-5.1)

**CMD:** [https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc771865(v=ws.11)](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc771865\(v=ws.11\))
