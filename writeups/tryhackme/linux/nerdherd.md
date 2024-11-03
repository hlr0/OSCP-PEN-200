---
description: https://tryhackme.com/room/nerdherd
---

# NerdHerd

## Nmap

As usual we kick off with a nmap scanning using -p- -sS -v switches to quickly scan all ports and show verberose information whilst scanning so we can start enumerating ports before the scan finished.

```
sudo nmap 10.10.237.0 -p- -sS -v

Discovered open port 21/tcp on 10.10.237.0
Discovered open port 139/tcp on 10.10.237.0
Discovered open port 445/tcp on 10.10.237.0
Discovered open port 22/tcp on 10.10.237.0
Discovered open port 1337/tcp on 10.10.237.0

Nmap scan report for 10.10.237.0
                                                                                                                                                                                                           
PORT     STATE SERVICE                                                                                                                                                                                                                     
21/tcp   open  ftp                                                                                                                                                                                                                         
22/tcp   open  ssh                                                                                                                                                                                                                         
139/tcp  open  netbios-ssn                                                                                                                                                                                                                 
445/tcp  open  microsoft-ds                                                                                                                                                                                                                
1337/tcp open  waste    
```

## FTP

Our first discovered port is FTP on TCP 21. I check for anonymous login and find we are allowed access. We have a directory called "pub" and in this we have a .png image file. Opening the file shows nothing interesting apart from the NerdHerd logo.

![](<../../../.gitbook/assets/image (310).png>)

### Testing for file upload

I tried to upload a file to FTP to see if we had anything here for a potential reverse shell however, we have no permission to perform this.

![](<../../../.gitbook/assets/image (311) (1).png>)
