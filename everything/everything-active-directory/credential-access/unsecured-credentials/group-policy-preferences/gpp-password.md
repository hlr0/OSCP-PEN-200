# 🔨 GPP Passwords

## What is it

{% embed url="https://docs.microsoft.com/en-us/security-updates/securitybulletins/2014/ms14-025" %}

For Windows Server 2012 R2 and previous before MS14-025 patch a scenario exists in where Administrators could create group policies would store credentials for users account policies or network drive mappings.

A password associated with a group policy such as a drive mapping would be stored in the 'Groups.xml' policy file in SYSVOL which is accessible to all domain users. The value would be stored under 'cpassword'.

The AES-32bit key for this was disclosed publicly and allows an attacker on the network to decrypt the key with publicly available tools which can potentially lead to privilege escalation on the domain.

MS14-025 will patch this vulnerability however, any policies created before the patch has been applied will need to be recreated as the patch cannot resolve policies created prior to the patch being applied.

## How to perform the attack

### Scenario

In this scenario we are an attacker that is on the internal network and has access to an account that can read the SYSVOL folder.

In the follow image we have downloaded the contents of the Replication folder in which in this instance contains the same information regarding Group Policy that SYSVOL will have.

![](<../../../../../.gitbook/assets/image (1543).png>)

From the above we notice we have downloaded 'Groups.xml'. From here we can open the XML file and look for the 'cpassword' value.

![](<../../../../../.gitbook/assets/image (1544).png>)

### Key Decryption

Now that we have the 'cpassword' value we can decrypt this with 'gpp-decrypt' which comes preinstalled with Kali Linux.

```
gpp-decrypt <value>
```

![](<../../../../../.gitbook/assets/image (1545) (1).png>)

We have obtained a password of 'GPPstillStandingStrong2k18'. As a side note from the XML file we have also obtained the username 'active.htb\SVC\_TGS'. I have covered this process for gaining access with these credentials in my write up for 'Active' on HackTheBox.

{% content-ref url="../../../../../writeups/hackthebox/active-directory/active.md" %}
[active.md](../../../../../writeups/hackthebox/active-directory/active.md)
{% endcontent-ref %}

## Further Techniques

{% content-ref url="./" %}
[.](./)
{% endcontent-ref %}
