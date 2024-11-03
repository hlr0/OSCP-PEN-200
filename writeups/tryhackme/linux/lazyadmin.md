# LazyAdmin

## Nmap

```
sudo nmap 10.10.43.217 -p- -sS -sV

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Port 80 lands us on to the Apache Ubuntu default installation page.

![](<../../../.gitbook/assets/image (1054).png>)

With nothing interesting in the page source code we can move onto directory enumeration with dirsearch.

```
python3 dirsearch.py -u http://10.10.43.217 -w /usr/share/seclists/Discovery/Web-Content/big.txt -r -t 60 --full-url 
```

dirsearch soon picks up the /content/ directory.

![](<../../../.gitbook/assets/image (1056).png>)

Browsing to the directory reveals SweetRice. Research on Google shows this is CMS software:[https://github.com/sweetrice](https://github.com/sweetrice)

![](<../../../.gitbook/assets/image (1055).png>)

Running dirsearch further on the /content/ directory with the raft-large-files.txt wordlist from seclists finds the changelog.txt page: [http://10.10.43.217/content/changelog.txt](http://10.10.43.217/content/changelog.txt)

```
python3 dirsearch.py -u http://10.10.43.217/content/ -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt -t 60 --full-url  
```

![](<../../../.gitbook/assets/image (1057).png>)

We are now aware we are running version 1.5.0 of SweetRice. Looking at the directory of /content/as/ we arrive at a login page.

![](<../../../.gitbook/assets/image (1059).png>)

The directory /content/inc/ also contains various files: [http://10.10.43.217/content/inc/](http://10.10.43.217/content/inc/) Of particular interest the the .sql backup in the mysql\_backup folder.

![](<../../../.gitbook/assets/image (1060).png>)

Downloading the file and running the strings command against it reveals login information:

![](<../../../.gitbook/assets/image (1061).png>)

Where we have a username of manager and a password has of: 42f749ade7f9e195bf475f37a44cafcb. This hash MD5 hash and was cracked using Hashcat on Windows with the rockyou.txt wordlist.

![](<../../../.gitbook/assets/image (1063).png>)

We now have credentials `manager:Password123`. We can then login to the CMS.

![](<../../../.gitbook/assets/image (1064).png>)

Looking at the Media Centre blade on the right we can upload files to the server. Initially i tried a standard PHP reverse shell and this was filtered out. Saving the Shell as a .php5 was not filtered out for upload.

![](<../../../.gitbook/assets/image (1065).png>)

At which point I set up a `netcat` listener and executed the uploaded file.

![](<../../../.gitbook/assets/image (1066).png>)

From here we have access to the home directory of 'itguy' in /home/itguy. This directory contains some interesting files namely the backup.pl file. The perl file is not writeable us but, looking at the containing script it executed /etc/copy.sh which we have write access to.

![](<../../../.gitbook/assets/image (1067).png>)

I then transferred over [pspy32](https://github.com/DominicBreuker/pspy/releases) to analyze if anything gets executed on a schedule in regards to either backup.pl or /etc/copy.sh. After waiting some time I assumed not.

![](<../../../.gitbook/assets/image (1069) (1).png>)

After checking sudo privileges we see that www-data can run backup.pl as any user without specifying a password.

![](<../../../.gitbook/assets/image (1070).png>)

From this we can replace the contents of /etc/copy.sh with a reverse shell knowing we can execute it with root privileges.

Checking the contents of /etc/copy.sh looks like it already contains a reverse shell...

From here I echo out the contents then replace it again this time with our attacking machine IP and port. Then execute /home/itguy/backup.pl using sudo under the context of root.

![](<../../../.gitbook/assets/image (1071).png>)

At which point we receive a reverse shell.

![](<../../../.gitbook/assets/image (1072).png>)
