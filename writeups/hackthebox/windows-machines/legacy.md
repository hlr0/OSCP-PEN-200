---
description: https://www.hackthebox.eu/home/machines/profile/2
---

# Legacy

![](<../../../.gitbook/assets/image (345).png>)

## Scanning & Enumeration

### Nmap

Starting off with a quick all port SYN scan we see we have SMB and RDP open.

```
nmap 10.10.10.4 -p- -sS -T4 -v

PORT     STATE  SERVICE
139/tcp  open   netbios-ssn
445/tcp  open   microsoft-ds
3389/tcp closed ms-wbt-server
```

We run a service version scan with `-sV` on the open ports. I slowed down the probe speed to `-T3` as we are dealing with so few ports. From the results this looks like the machine is Windows XP.

```
nmap 10.10.10.4 -p 139,445,3389 -T3 -sS -sV

PORT     STATE  SERVICE       VERSION
139/tcp  open   netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open   microsoft-ds  Microsoft Windows XP microsoft-ds
3389/tcp closed ms-wbt-server

Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp
```

We can confirm our results using the `--script smb-os-discovery` script.

```
nmap 10.10.10.4 --script smb-os-discovery

Host script results:
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2020-11-20T00:45:42+02:00
```

Since we have SMB open and is likely the attack vector lets check the exact version before researching exploits.

```
nmap --script smb-protocols 10.10.10.4 -Pn

Host script results:
| smb-protocols: 
|   dialects: 
|_    NT LM 0.12 (SMBv1) [dangerous, but default]
```

We can take the information we now have and use Google to search for any relevant exploits.

![](<../../../.gitbook/assets/image (346).png>)

As you can see we have a few potential exploits in which we could try. Also looking at the related searches at the bottom of the page will give us results from other peoples searches which I find is a great way to find relevant information.

![](<../../../.gitbook/assets/image (347) (1).png>)

MS08-067 is of interest.

## Exploitation

### MS08-067

We can start `metasploit` with `msfconsole` command and search for MS08-067 to see if we have any modules for it.

![](<../../../.gitbook/assets/image (348).png>)

We have a module and select it with `use 0`. Once selected we can use `show options` command to show what options we need to set.

![](<../../../.gitbook/assets/image (349) (1).png>)

* RHOSTS - Remote host IP
* LHOST - Attacking machine IP (You can set the interface name instead)
* LPORT - Local port to use. Select any port you like. The default 4444 is usually good enough.

We can then execute the exploit with the `run` command.

![](<../../../.gitbook/assets/image (350) (1).png>)

Once connected we can get the session GUID to confirm completed. From here we can drop into a system shell using the `shell` command.

![](<../../../.gitbook/assets/image (351).png>)

Once in we can check if we can access the Administrator Desktop and can view the root.txt flag. We can also browse the Documents and Settings folder for the other user on the system to grab the user.txt flag from the desktop.

### MS17-010

When we was looking for exploits earlier we come across a fair few mentions for MS17\_010.

{% embed url="https://docs.microsoft.com/en-us/security-updates/SecurityBulletins/2017/ms17-010" %}

Under the first paragraph we see the follow mentioned.

![](<../../../.gitbook/assets/image (352) (1).png>)

As we are running SMBv1 as discovered by `nmap` earlier using the `smb-protocols` script. Lets look further into this.

Again we start `msfconsole` and search for MS17\_10. We can start with using the auxiliary scanner first to determine if the host is vulnerable.

![](<../../../.gitbook/assets/image (353).png>)

We can also use the show info command on the auxiliary scanner module to see more information regarding what this checks for and references for further reading.

![](<../../../.gitbook/assets/image (354).png>)

We can also check if the host is vulnerable to this exploit using `nmap`.

```
nmap -Pn --script smb-vuln-ms17-010 -p 445 10.10.10.4

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
```

As we have determined the host is likely vulnerable we can now attempt to exploit it. From our search results I am going to use the module "exploit/windows/smb/ms17\_010\_psexec". We set the relevant options again and run the exploit.

![](<../../../.gitbook/assets/image (355).png>)
