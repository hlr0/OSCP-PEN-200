---
description: https://attack.mitre.org/techniques/T1070/001/
---

# Clear Windows Event Logs

**ATT\&CK ID:** [T1070.001](https://attack.mitre.org/techniques/T1070/001/)

**Permissions Required:**

**Description**

Adversaries may clear Windows Event Logs to hide the activity of an intrusion. Windows Event Logs are a record of a computer's alerts and notifications. There are three system-defined sources of events: System, Application, and Security, with five event types: Error, Warning, Information, Success Audit, and Failure Audit.

These logs may also be cleared through other mechanisms, such as the event viewer GUI or PowerShell.

\[[Source\]](https://attack.mitre.org/techniques/T1070/001/)

## **Techniques**

### Metasploit

This meterpreter command wipes logs from Application,System and Security logs.

```bash
# Meterpreter
clearev
```

![](<../../../../.gitbook/assets/image (559) (2).png>)

### PowerShell

Performing this command leaves an event for the logs being cleared.

```powershell
#Clear Application,Security and System Logs
Clear-Eventlog -LogName Application,Security,System

# Utilize PowerShell with Wevtutil to clear all logs from the system
wevtutil el | Foreach-Object {wevtutil cl $_}
```

![](../../../../.gitbook/assets/EventLog.png)

### **Wevtutil**

```
# Clear all logs on the system (cmd)
 for /F "tokens=*" %1 in ('wevtutil.exe el') DO wevtutil.exe cl "%1"

# Clear select logs
wevtutil cl system
wevtutil cl application
wevtutil cl security
```

![](../../../../.gitbook/assets/wevtutil-clear-all-logs.png)

## **Mitigation**

* Automatically forward events to a log server or data repository to prevent conditions in which the adversary can locate and manipulate data on the local system. When possible, minimize time delay on event reporting to avoid prolonged storage on the local system.
* Protect generated event files that are stored locally with proper permissions and authentication and limit opportunities for adversaries to increase privileges by preventing Privilege Escalation opportunities.

## **Further Reading**

**Wevtutil:** [https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/wevtutil](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/wevtutil)

**Clear-EventLog:** [https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/clear-eventlog?view=powershell-5.1](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/clear-eventlog?view=powershell-5.1)
