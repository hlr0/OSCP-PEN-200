---
description: https://www.cyberseclabs.co.uk/labs/info/Weak/
---

# Weak

## Scanning and Enumeration

### Nmap

Running a basic `nmap` scan with the `-sV` switch to scan service version against the target machines open ports we get the following:

```
sudo nmap 172.31.1.11 -p- -sV
Starting Nmap 7.91 ( https://nmap.org ) at 2020-12-08 11:34 EST
Stats: 0:00:58 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 95.07% done; ETC: 11:35 (0:00:03 remaining)
Nmap scan report for 172.31.1.11
Host is up (0.032s latency).
Not shown: 65519 closed ports
PORT      STATE SERVICE            VERSION
21/tcp    open  ftp                Microsoft ftpd
80/tcp    open  http               Microsoft IIS httpd 7.5
135/tcp   open  msrpc              Microsoft Windows RPC
139/tcp   open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds       Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
554/tcp   open  rtsp?
2869/tcp  open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
3389/tcp  open  ssl/ms-wbt-server?
5357/tcp  open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
10243/tcp open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49152/tcp open  msrpc              Microsoft Windows RPC
49153/tcp open  msrpc              Microsoft Windows RPC
49154/tcp open  msrpc              Microsoft Windows RPC
49155/tcp open  msrpc              Microsoft Windows RPC
49161/tcp open  msrpc              Microsoft Windows RPC
49162/tcp open  msrpc              Microsoft Windows RPC
```

We can use the `smb-os-discovery` script to confirm the Operating system as port 445 is open.

```
nmap 172.31.1.11 --script=smb-os-discovery -p 445

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-os-discovery: 
|   OS: Windows 7 Ultimate 7601 Service Pack 1 (Windows 7 Ultimate 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1
|   Computer name: Weak
|   NetBIOS computer name: WEAK\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2020-12-08T08:34:33-08:00
```

As from the above script we are dealing with Windows 7 Ultimate.

### Port 445 (SMB)

As Port 455 is open we can start looking here. I firstly run generic commands to check for null authentication. Unfortunately I was unable to proceed with this.

![](<../../../.gitbook/assets/image (446).png>)

I was however, able to log in with `rpcclient`. No interesting commands was available to me so I will shelf this possible avenue until later.

### Port 80 (HTTP)

We have HTTP on port and the default root page takes us to the II7 welcome page.

![](<../../../.gitbook/assets/image (447).png>)

I proceeded to run `nikto` and `feroxbuster` against this to determine if we can find any other directories.

![](<../../../.gitbook/assets/image (448).png>)

### Port 21 (FTP)

We use `nmap` to check if we have anonymous access to the FTP server.

```
nmap 172.31.1.11 --script=ftp-anon -p 21                                                                                                                                                                                      
                                                                                                                                                                                                             
PORT   STATE SERVICE                                                                                                                                                                                                                       
21/tcp open  ftp                                                                                                                                                                                                                           
| ftp-anon: Anonymous FTP login allowed (FTP code 230)                                                                                                                                                                                     
| 04-11-20  12:32PM       <DIR>          aspnet_client                                                                                                                                                                                     
| 04-10-20  01:30AM                  689 iisstart.htm                                                                                                                                                                                      
|_04-10-20  01:30AM               184946 welcome.png                                                                                                                                                                                                                   
```

`nmap` has confirmed we can log into the FTP with anonymous credentials and has even shows us the available contents. We can now manually log in so we can browse the FTP server.

![](<../../../.gitbook/assets/image (449) (1).png>)

When trying to run the `dir` command to list contents we are given the error '501 Server cannot accept Argument'.

We can attempt to login to the FTP again this time with the `-p` switch for passive mode.

![](<../../../.gitbook/assets/image (450) (1).png>)

We have now logged in and can run the `dir` command to list directory contents. Browsing through the aspnet\_client directory we get version information we can possibly use later. The directories shown below did not hold any files or folders.

![](<../../../.gitbook/assets/image (451).png>)

Going back to the root of the FTP directory we can test for file upload with the `put` command.

![](<../../../.gitbook/assets/image (452).png>)

As we have confirmed upload on the directory we can then attempt to confirm if we can read the file with the `curl` command.

```
curl http://172.31.1.11/<FileName>
```

![](<../../../.gitbook/assets/image (453).png>)

The contents of the uploaded file has been read by curl. We could also browse to this text file in the web browser.

![](<../../../.gitbook/assets/image (454) (1).png>)

## Low Privilege User Access

From this point I tried upload multiple ASP and ASPX reverse shells and could not get a hit. I tried a PHP web shell and could not get anything.

I then come across a CMD web shell which after uploading with the `put` command in FTP worked really well for me.

{% embed url="https://github.com/tennc/webshell/blob/master/fuzzdb-webshell/asp/cmd.aspx" %}

Download the file and upload it to the FTP. When you browse to it you should see the following:

![](<../../../.gitbook/assets/image (455).png>)

Remember the `/c` tells `cmd.exe` to execute the following command. This shell will not work correctly without that switch.

We can now check who we are running as.

![](<../../../.gitbook/assets/image (456) (1).png>)

From here I tried downloading a reverse shell directly onto the machine with certutil.exe but was given access denied.

![](<../../../.gitbook/assets/image (457).png>)

I decided to look around the file system at this point and come across a README inside a non default directory called 'Development'.

![](<../../../.gitbook/assets/image (458).png>)

{% hint style="danger" %}
Ignore the .exe file. This image was taken post compromise and the .exe is not actually part of the machine.
{% endhint %}

We can then read the contents of the README.txt file.

![](<../../../.gitbook/assets/image (459) (1).png>)

We have a password but we need to identify the account. We can check local accounts on the machine with net user.

![](<../../../.gitbook/assets/image (460).png>)

The user 'Web Admin' is of interest. We can take the gathered credentials and try using these against the machine.

We can confirm if the credentials work with `crackmapexec`.

```
crackmapexec smb 172.31.1.11 -u 'Web Admin' -p Password
```

## Access as High Value Target

![](<../../../.gitbook/assets/image (461).png>)

Now that we have confirmed credentials I tried throwing them against Impacket's `smbexec.py` and `wmiexec.py` to see if we can gain shell.

![](<../../../.gitbook/assets/image (462).png>)

I then tried against `psexec.py` and was able to gain shell as 'NT Authority\System'.

![](<../../../.gitbook/assets/image (463).png>)

From here we can grab the hashes from both the Administrator desktop and the user desktop.
