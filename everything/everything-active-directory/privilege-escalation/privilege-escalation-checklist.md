# Privilege Escalation Checklist

## Automated Tools

* **JAWS:** [https://github.com/411Hall/JAWS](https://github.com/411Hall/JAWS)
* **Metasploit:** `multi/recon/local_exploit_suggester`&#x20;
* **PowerUp:** [https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc](https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc)
* **Seatbelt:** [https://github.com/GhostPack/Seatbelt](https://github.com/GhostPack/Seatbelt)
* **Watson:** [https://github.com/rasta-mouse/Watson](https://github.com/rasta-mouse/Watson)
* **Windows Exploit Suggester:** [https://github.com/AonCyberLabs/Windows-Exploit-Suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester)
* **WinPEAS:** [https://github.com/carlospolop/PEASS-ng/releases](https://github.com/carlospolop/PEASS-ng/releases)

## System Information

Check Installed OS and architecture

{% tabs %}
{% tab title="CMD" %}
```
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
```
{% endtab %}

{% tab title="PS" %}
```
Get-ComputerInfo -property 'WindowsProductName', 'OsVersion', 'OsArchitecture'
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" -Name ReleaseId
```
{% endtab %}
{% endtabs %}

Get Installed updates

{% tabs %}
{% tab title="CMD" %}
```
systeminfo | find ": KB"

wmic qfe get Caption,Description,HotFixID,InstalledOn
```
{% endtab %}

{% tab title="PS" %}
```
get-wmiobject -class win32_quickfixengineering
```
{% endtab %}
{% endtabs %}

List environment variables

{% tabs %}
{% tab title="CMD" %}
```
set
```
{% endtab %}

{% tab title="PS" %}
```
Get-ChildItem Env: | ft Key,Value
```
{% endtab %}
{% endtabs %}

List local and network drives

{% tabs %}
{% tab title="CMD" %}
```
wmic logicaldisk get deviceid, volumename, description
```
{% endtab %}

{% tab title="PS" %}
```
get-psdrive -psprovider filesystem
```
{% endtab %}
{% endtabs %}

View Domain Controllers

{% tabs %}
{% tab title="CMD" %}
```
systeminfo | findstr /B /C:"Domain"
```
{% endtab %}
{% endtabs %}

## Network

Get interface and network configuration

{% tabs %}
{% tab title="CMD" %}
```
ipconfig /all
```
{% endtab %}

{% tab title="PS" %}
```
Get-NetIPConfiguration | Select-Object -Property InterfaceAlias, IPv4Address
```
{% endtab %}
{% endtabs %}

Print routing table

{% tabs %}
{% tab title="CMD" %}
```
route print
```
{% endtab %}
{% endtabs %}

List active connections

{% tabs %}
{% tab title="CMD" %}
```
netstat -ano
```
{% endtab %}

{% tab title="PS" %}
```
Get-NetTCPConnection
```
{% endtab %}
{% endtabs %}

Show Firewall state and configuration

{% tabs %}
{% tab title="CMD" %}
```
netsh firewall show state
netsh firewall show config
```
{% endtab %}
{% endtabs %}

List network drives

{% tabs %}
{% tab title="CMD" %}
```
net share
```
{% endtab %}

{% tab title="PS" %}
```
Get-SMBMapping
```
{% endtab %}
{% endtabs %}

View DNS cache

{% tabs %}
{% tab title="CMD" %}
```
ipconfig /displaydns
```
{% endtab %}
{% endtabs %}

## Users and Groups

Get current user

{% tabs %}
{% tab title="CMD" %}
```
whoami
net user %username%
```
{% endtab %}
{% endtabs %}

List all users

{% tabs %}
{% tab title="CMD" %}
```
net user
whoami /all
```
{% endtab %}

{% tab title="PS" %}
```
Get-LocalUser | ft Name,Enabled,LastLogon,Description
```
{% endtab %}
{% endtabs %}

Get details about a specific user

{% tabs %}
{% tab title="CMD" %}
```
net user <user>
```
{% endtab %}
{% endtabs %}

View password policy

{% tabs %}
{% tab title="CMD" %}
```
net accounts
```
{% endtab %}
{% endtabs %}

Get local groups

{% tabs %}
{% tab title="CMD" %}
```
net localgroup
```
{% endtab %}
{% endtabs %}

## Services

Get running services

{% tabs %}
{% tab title="CMD" %}
```
wmic service get Caption,StartName,State,pathname
```
{% endtab %}

{% tab title="CMD" %}
```
net start
```
{% endtab %}
{% endtabs %}

List unquoted service binaries

```
wmic service get name,pathname,displayname,startmode | findstr /i auto | findstr /i /v "C:\Windows\\" | findstr /i /v """
```

##

## World Writeable Folders

```
C:\Windows\System32\Microsoft\Crypto\RSA\MachineKeys
C:\Windows\System32\spool\drivers\color
C:\Windows\Tasks
C:\Windows\tracing
C:\Windows\Temp
C:\Users\Public
```

## Privilege Escalation Specific

Unquoted service paths

If value returned is `AlwaysInstallElevated REG_DWORD 0x1` A malicious MSI can be used to install with elevated permissions from a standard privileged account.

```
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated
```

### Check Sticky Notes for passwords

```
C:\Users\<user>\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite
```

## Search File System for passwords and files of interest

### Search for passwords

```
findstr /si password *.txt
findstr /si password *.xml
findstr /si password *.ini
findstr /si pass *.txt
findstr /si pass *.xml
findstr /si pass *.ini

#Find all those strings in config files.
dir /s *pass* == *cred* == *vnc* == *.config*
```

If current user can read Event Logs then get the latest PowerShell commands run on the system

```
Get-EventLog -LogName 'Windows PowerShell' -Newest 100 | Select-Object -Property * 
```

### Recycle Bin

```
cd 'c:\$recycle.bin\<User SID>'
dir /A
```
