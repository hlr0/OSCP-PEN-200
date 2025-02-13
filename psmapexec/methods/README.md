# Methods

PsMapExec currently supports the following methods:

* Inject
* IPMI
* Kerberoast
* MSSQL (New addition, not yet feature complete)
* SMB
* SessionHunter
* RDP
* VNC
* WinRM
* WMI

## Command Execution Methods

The following methods support command execution and running modules on target systems:

* MSSQL (Need SYSADMIN on Instance)
* SMB
* SessionHunter (WMI)
* WinRM
* WMI

### Authentication Types

When  `-Command` and `-Module` are omitted, PsMapExec will simply check the provided or current user credentials against the specified target systems for administrative access over the specified method.

```powershell
# Current user
PsMapExec -Targets All -Method [Method]

# With Password
PsMapExec -Targets All -Method [Method] -Username [Username] -Password [Password]

# With Hash
PsMapExec -Targets All -Method [Method] -Username [Username] -Hash [RC4/AES256]

# With Ticket
PsMapExec -Targets All -Method [Method] -Ticket [doI.. OR Path to ticket file]

# Local Authentication (WMI and MSSQL only)
PsMapExec -Targets All -Method WMI -Username Administrator -Password Password -LocalAuth
```

### Command Execution

All currently supported command execution methods support the `-Command`  parameter. The command parameter can be appended to the above Authentication Types to execute given commands as a specified or current user.

{% code overflow="wrap" %}
```powershell
PsMapExec -Targets All -Method [Method] -Command [Command]
```
{% endcode %}

### Module Execution

All currently supported command execution methods support the `-Module`  parameter. The module parameter can be appended to the Authentication Types to execute given modules as a specified or current user.

```
PsMapExec -Targets All -Method [Method] -Module [Module]
```

For supported modules and syntax visit the link below

{% content-ref url="../modules/" %}
[modules](../modules/)
{% endcontent-ref %}

## GenRelayList / SMB Signing

The GenRelayList method checks targets SMB signing requirements.

{% content-ref url="genrelaylist-smb-signing.md" %}
[genrelaylist-smb-signing.md](genrelaylist-smb-signing.md)
{% endcontent-ref %}

## IPMI

The IPMI method attempts to retrieve IPMI hashes from vulnerable servers

{% content-ref url="ipmi.md" %}
[ipmi.md](ipmi.md)
{% endcontent-ref %}

## Kerberoast

Performs kerberoasting against the current or specified domain.

{% content-ref url="kerberoast.md" %}
[kerberoast.md](kerberoast.md)
{% endcontent-ref %}

## SessionHunter

The SessionHunter method filters targets by those which are likely to have administrative or privileged users credentials in memory and for which we have administrative access to. This is the ideal way of filtering down to a small number of system for which to extract credentials from to escalate privileges further.

{% content-ref url="session-hunter.md" %}
[session-hunter.md](session-hunter.md)
{% endcontent-ref %}

## Spray

PsMapExec supports password and hash spraying as well as some additional parameters for spraying accounts as passwords and empty password values.&#x20;

{% content-ref url="spray.md" %}
[spray.md](spray.md)
{% endcontent-ref %}

## VNC

The VNC method is used to simply check if a VNC server has "NoAuth" set which means we can connect to the remote system without providing a username or password.

```
PsMapExec -Targets [Targets] -Method VNC
```
