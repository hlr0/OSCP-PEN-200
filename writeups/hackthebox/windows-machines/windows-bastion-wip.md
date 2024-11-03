---
description: https://www.hackthebox.eu/home/machines/profile/186
---

# Bastion

## Nmap

We can start of by quickly checking for alive ports with `nmap -p- -T4` and then after the results have returned I executed a more intensive scan on the ports that I believe to be interesting

```
nmap 10.10.10.134 -p 22,445,5985,47001 -T4 -A

PORT      STATE SERVICE      VERSION
22/tcp    open  ssh          OpenSSH for_Windows_7.9 (protocol 2.0)
| ssh-hostkey: 
|   2048 3a:56:ae:75:3c:78:0e:c8:56:4d:cb:1c:22:bf:45:8a (RSA)
|   256 cc:2e:56:ab:19:97:d5:bb:03:fb:82:cd:63:da:68:01 (ECDSA)
|_  256 93:5f:5d:aa:ca:9f:53:e7:f2:82:e6:64:a8:a3:a0:18 (ED25519)
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OSs: Windows Server 2008 R2 - 2012, Windows; CPE: cpe:/o:microsoft:windows
                                                                                                                                                                                                                                           
Host script results:                                                                                                                                                                                                                       
|_clock-skew: mean: -39m37s, deviation: 1h09m15s, median: 21s                                                                                                                                                                              
| smb-os-discovery:                                                                                                                                                                                                                        
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)                                                                                                                                                              
|   Computer name: Bastion                                                                                                                                                                                                                 
|   NetBIOS computer name: BASTION\x00                                                                                                                                                                                                     
|   Workgroup: WORKGROUP\x00                                                                                                                                                                                                               
|_  System time: 2020-10-08T15:03:22+02:00                                                                                                                                                                                                 
| smb-security-mode:                                                                                                                                                                                                                       
|   account_used: guest                                                                                                                                                                                                                    
|   authentication_level: user                                                                                                                                                                                                             
|   challenge_response: supported                                                                                                                                                                                                          
|_  message_signing: disabled (dangerous, but default)                                                                                                                                                                                     
| smb2-security-mode:                                                                                                                                                                                                                      
|   2.02:                                                                                                                                                                                                                                  
|_    Message signing enabled but not required                                                                                                                                                                                             
| smb2-time:                                                                                                                                                                                                                               
|   date: 2020-10-08T13:03:21                                                                                                                                                                                                              
|_  start_date: 2020-10-08T12:41:1
```

## Legion

Since we have a few ports open and some SMB shares I will take this opportunity to run `Legion` against this machine. `Legion` comes pre-installed on Kali and can be run with using `sudo legion`. Once started add the host IP address into legion and fire away with default settings.

![Adding the target to legion](<../../../.gitbook/assets/image (217).png>)

Wait a short while for `legion` to finish running and we can check the results. If we look under the smbenum tab (445/tcp) we can see some interesting results regarding SMB shares.

![smbenum on port 445](<../../../.gitbook/assets/image (218) (1).png>)

From the above output I connected to the Backups share using `smbclient`. We can see from the output above we have read access to this share. When specifying the `-N` switch and not defining a username we are attempting to logon to SMB with a null session.

![](<../../../.gitbook/assets/image (219) (1).png>)

I downloaded the note.txt file with the `get` command which contained the following information:

![note.txt](<../../../.gitbook/assets/image (220) (1).png>)

When we inspect the shares we notice that we come across some VHD files. Initially I tried downloading these files over the VPN connection however, as per the note above this was much too slow.

![](<../../../.gitbook/assets/image (221).png>)

## Mounting VHD files to Linux

The next best option for us given the speed constraints and the size of the backup VHD's would be to mount them to our system so we can browse the files within.

First ensure the following tools are installed:

```
apt-get install libguestfs-tools
apt-get install cifs-utils
```

After this create a directory in the /mnt/ directory using the `mkdir` command.

![](<../../../.gitbook/assets/image (223) (1).png>)

We can then mount the remote share to the directory we just created.

```
sudo mount -t cifs -o username=NULL //10.10.10.134/Backups/WindowsImageBackup  /mnt/bastion -o rw
```

When prompted for a password just hit enter so we can authenticate with a NULL session. As you can see we are now browsing the attached SMB share from our mount point. Make a note of the full path of the VHD file as we will be mounting the VHD directly next.

![](<../../../.gitbook/assets/image (224) (1).png>)

Now create a new mount point ready for the VHD to be mounted. I had some issues with using /mnt/ directory for this as I was not running directly as root. I had better results mounting in a new folder in my home directory.

```
mkdir /home/kali/Desktop/files
```

After the new directory has been created run `guestmount` with the following command. Replacing both paths where appropriate.

```
guestmount --add "/mnt/bastion/L4mpje-PC/Backup 2019-02-22 124351/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd" --inspector --ro /home/kali/Desktop/files -v
```

If you have issues with `libguestfs-tools` process dying you may need to purge the install for it and then reinstall and then perform a `apt-get update && apt-get` upgrade on the system to get the tool to work correctly. I had two do this twice on two different machines running the latest version of Kali.

## User

We are going to be going after the Security Account Manager (SAM) file. Usually this file is protected from access when the operating system is running and can be further protected by disk encryption from physical attacks. As we have direct access to the VHD we simply have read access to the system files.

We can use a tool called `samdump2` which comes pre-installed on Kali to extract the local account hashes from the SAM file. Navigate to the following path to find the SAM file `/Windows/System32/config` we can then run the following command to extract the hashes to a specified file s`amdump2 SYSTEM SAM > /home/kali/Desktop/hash.txt`

![](<../../../.gitbook/assets/image (227) (1).png>)

I have removed the disabled accounts from the hash.txt file. You can either run this against John the Ripper to attempt to crack the hashes or in this instance I will be using [http://crackstation.net](https://crackstation.net) to find the password. with NTLM hashes you always attempt to crack the second hash in the line.

![](<../../../.gitbook/assets/image (228) (1).png>)

We have the following password "bureaulampje".

From the `nmap` results earlier we recall having SSH running on port 22. We can attempt to login with `ssh L4mpje@10.10.10.134` with the password we cracked.

Once logged in we can grab the user.txt flag.

![](<../../../.gitbook/assets/image (229).png>)

## Privilege Escalation

For privilege escalation we will be looking into the installed programs to identify anything unusual. In `C:\Program Files (x86)` we can see the directory mRemoteNG directory. After researching the application name on Google. mRemoteNG is a connection manager for various connection protocols such as SSH, RDP and Telnet to name a few.

When searching for a relevant exploit I came across the following post:

{% embed url="https://cosine-security.blogspot.com/2011/06/stealing-password-from-mremote.html" %}

The post mentions a `metasploit` module.. Going back to Google and searching for other ways to decrypt the password we come across a python script that can decrypt for us:

{% embed url="https://github.com/haseebT/mRemoteNG-Decrypt" %}

Download the python script and store it for later. We need to now go and hunt down the password string.

The blog post from cosine-security mentions the passwords being stored in an XML file in AppData. Going back to the SSH connection we can start looking about AppData. We come to this interesting file which contains the information we need. Reading through the first line we can see stored information for the Administrator account. Take the password string and copy it so we can run it through the python script downloaded from github.

![confCons.xml](<../../../.gitbook/assets/image (231) (1).png>)

Running the python script:

## Root Flag

![](<../../../.gitbook/assets/image (232) (1).png>)

We can take this password and login to SSH with it under the Administrator account. From here we can grab the root.txt flag.

![](<../../../.gitbook/assets/image (233).png>)
