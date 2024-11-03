---
description: https://tryhackme.com/room/gallery666
---

# Gallery

## Nmap

```
nmap 10.10.40.100 -p- -sS -sV

PORT     STATE    SERVICE VERSION
53/tcp   filtered domain
80/tcp   open     http    Apache httpd 2.4.29 ((Ubuntu))
8080/tcp open     http    Apache httpd 2.4.29 ((Ubuntu))
```

Starting out on port 80 we have the Apache2 Default Page.

![](<../../../.gitbook/assets/image (379).png>)

Running `feroxbuster` against the target web server and we see some results.

```
feroxbuster -u http://10.10.40.100 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
```

![](<../../../.gitbook/assets/image (1120).png>)

Checking the gallery directory we are redirected to `/gallery/login.php`.

![](<../../../.gitbook/assets/image (877).png>)

Attempting a login with a simple set of credentials, we check the request in ZAP web proxy and notice a SQL error code has been returned in response to the logon.

![](<../../../.gitbook/assets/image (1527).png>)

Knowing SQL is running on the back end we can try some simple SQL injection on the login page.

Logging in with the SQL injection `' OR 1=1 -- -` for the username and password proves successful.

![](<../../../.gitbook/assets/image (1378).png>)

**Note:** We can also use `sqlmap` with the raw proxy request to enumerate the database and pull the admin hash.

```
sqlmap -r ~/Desktop/request.raw --batch -D gallery_db -T users --columns username,password --dump
```

![](<../../../.gitbook/assets/image (3) (1) (2).png>)

Back to the logged in page we managed to bypass. We notice we are able to upload a new profile picture to our current administrative user.

![](<../../../.gitbook/assets/image (2049).png>)

As we know PHP is running on the webserver we will ideally upload a PHP reverse shell as the avatar.

**RevShells:** [https://www.revshells.com/](https://www.revshells.com/)

I opted for the PHP PentestMonkey shell.

After uploading the shell we find it triggers our `netcat` receiver instantly.

**Note:** If for anyone reason this does not trigger we can right click where the avatar is supposed to be in the top right and "Open image in a new tab" to trigger.

![](<../../../.gitbook/assets/image (159) (1).png>)

Looking through the /var/backups directory we find the directory "mike\_home\_backup".

![](<../../../.gitbook/assets/image (26) (1) (2).png>)

Within the contents we find that `.bash_history` is readable by us which, contains a potential password.

![](<../../../.gitbook/assets/image (2068).png>)

Using the credentials (altered slightly due to a type in the history file) we are able to `su` over to _mike_.

![](<../../../.gitbook/assets/image (231).png>)

Where we can then read the `user.txt` flag in Mike's home directory.

![](<../../../.gitbook/assets/image (1129).png>)

Running `sudo -l` we see mike is able to run bash on `/opt/rootkit.sh` as **root** without requiring a password.

![](<../../../.gitbook/assets/image (273).png>)

Reading the contents of `/opt/rootkit.sh` we see we will be prompted for commands, read looks to be the most promising due to hit executing `nano`.

![](<../../../.gitbook/assets/image (302).png>)

Looking at GTFOBins we see we are able to escape nano to run system commands.

**GTFOBins:** [https://gtfobins.github.io/gtfobins/nano/](https://gtfobins.github.io/gtfobins/nano/)

![](<../../../.gitbook/assets/image (1566).png>)

**Note:** I was unable to complete the privilege escalation initially as the `nc` shell I was currently using was not stable enough to even work over TTY. `Socat` is installed on the target system and is highly recommended to get a reverse shell with `Socat` as the user mike to complete the next steps.

When ready we execute /opt/rootkit.sh as root.

```
sudo -u root /bin/bash /opt/rootkit.sh
```

Then enter "read" into the interactive prompt where we will move over to a root `nano` session. Pressing `CTRL + R and CTRL + X` on the keyboard will prompt for a command to run in nano.

![](<../../../.gitbook/assets/image (415).png>)

Entering the command below will escape into a root shell and all us to grab the **root** flag.

```
reset; sh 1>&0 2>&0
```

![](<../../../.gitbook/assets/image (142).png>)
