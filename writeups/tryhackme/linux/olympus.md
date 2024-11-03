---
description: https://tryhackme.com/room/olympusroom
---

# Olympus

## Nmap

```
sudo nmap 10.10.46.71 -p- -sS -sV                                          

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Web Server

**Note:** Add olympus.thm to `/etc/hosts`.

Browsing to the web server on port 80 after amending the hosts file, we are presented with the page shown below.

![](<../../../.gitbook/assets/image (7) (3).png>)

Using `feroxbuster` to force browse the directories on the target system we get a valid hit for `/~webmaster`.

![](<../../../.gitbook/assets/image (11) (2) (1).png>)

### Victor CMS - SQLi

From here we are taken to a new page that appears to be running Victor CMS.

![](<../../../.gitbook/assets/image (1) (3) (1).png>)

A simple search with `searchsploit` shows multiple known issues. After going through some of the vulnerabilities I oped for the "cat\_id" option as this does not require any authentication.

![](<../../../.gitbook/assets/image (26) (2).png>)

**Attack vector:** [https://www.exploit-db.com/exploits/48485](https://www.exploit-db.com/exploits/48485)

![](<../../../.gitbook/assets/image (8) (1) (1).png>)

Using the PoC linked above we perform the request and capture with `Burpsuite`. Saving the raw request in the process.

![](<../../../.gitbook/assets/image (10) (2).png>)

### SQLmap

With the saved request we utilize it with `sqlmap`. Using the --batch option we discover the olympus database and the "chats" table.&#x20;

Together with this information we then dump the table.

```
sqlmap -r ~/Desktop/request.raw --batch -D olympus -T chats --dump
```

Where we retrieve three password hashes for the accounts:

* prometheus
* zeus
* root

![](<../../../.gitbook/assets/image (32) (2).png>)

### Flag #1

Dumping all tables in the olympus database also reveals the first flag under the "flag" table.

![](<../../../.gitbook/assets/image (2) (3).png>)

### Hash cracking

Running john against the captures hashes we soon get a single hit on the first hash for the prometheus.

![](<../../../.gitbook/assets/image (22) (2) (1).png>)

### Chat.Olympus.thm

From the remaining information dumped from the olympus database we also discver the virtual host "chat.olympus.thm". Browsing to this we find the following login page.

![](<../../../.gitbook/assets/image (15) (3).png>)

Where using prometheus' credentials cracked earlier we are able to authenticate. Looking at the existing message log we see we have the ability to upload files, however we see that uploaded file names are randomized.

![](<../../../.gitbook/assets/image (34).png>)

Firstly, we upload a PHP reverse shell.

![](<../../../.gitbook/assets/image (18) (3) (1).png>)

Then looking back at sqlmap we find the database stores the name of the file after it has been randomized.

![I](<../../../.gitbook/assets/image (21) (2).png>)

### Shell as www-data

In the chat application we see mention of an "uploads" folder. Using this information combined with the name of the shell we have uploaded, we can browse and trigger our shell.

Browse to:[http://chat.olympus.thm/uploads/b8b91888be70b2a182bffb24fd12441a.php](http://chat.olympus.thm/uploads/b8b91888be70b2a182bffb24fd12441a.php)

Where we obtain a shell as _www-data_.

![](<../../../.gitbook/assets/image (36).png>)

### Flag #2

The next flag can be found in `/home/zeus/user.flag`.

![](<../../../.gitbook/assets/image (30).png>)

### Privilege Escalation #1

For privilege escalation linpeas.sh was run against the target system, linpeas identified  where the binary `/usr/bin/cputils` has the SUID bit set for the user _zeus_.

![](<../../../.gitbook/assets/image (13) (1) (1).png>)

Running the strings command against the binary we see what the purpose of the binary is.

![](<../../../.gitbook/assets/image (5) (4).png>)

Running the binary itself, we can run within the context of the user _zeus_. Knowing this we copy the `/home/zeus/.ssh/id_rsa` key to the `/tmp/` directory.

![](<../../../.gitbook/assets/image (13) (3).png>)

The `id_rsa` key is then copied to the attacking system. When attempting to use the key file we are prompted for a password.

![](<../../../.gitbook/assets/image (27) (2).png>)

`ssh2john` can be utilized to hash the `id_rsa` file and then cracked to reveal the password.

```
/usr/bin/ssh2john ~/Desktop/id_rsa >> ~/Desktop/id_rsa.hash
```

![](<../../../.gitbook/assets/image (29) (2).png>)

### SSH as zeus

After cracking the hash and obtaining the password we are then able to login as _zeus_.

![](<../../../.gitbook/assets/image (20) (2).png>)

Manual enumeration of the web directories reveals a directory with a randomized name and within, a PHP file with a randomized name.

![](<../../../.gitbook/assets/image (24) (2).png>)

Viewing the contents of `VIGQFQNYOST.php`.

![](<../../../.gitbook/assets/image (17) (3).png>)

Where we can also browse to the same location and file:

![](<../../../.gitbook/assets/image (16) (2) (1).png>)

However, I was unable to proceed using the root shell shown in the image below:

![](<../../../.gitbook/assets/image (23) (2) (1) (1).png>)

### Privilege Escalation #2

A further look through at the PHP code and we can see we can call a binary file to spawn a root shell.

![](<../../../.gitbook/assets/image (31).png>)

```
uname -a; w; /lib/defended/libc.so.99
```

![](<../../../.gitbook/assets/image (33) (2).png>)

### Flag #3

After spawning the root shell we can then grab the 3rd flag.

![](<../../../.gitbook/assets/image (19) (2).png>)

### Flag #4

For the 4th flag we are advised by the room creator to search for it within the root shell. We can utilize grep with the known syntax of the previous flags to find it.

```
grep -irl flag{ /etc/
```

![](<../../../.gitbook/assets/image (4) (3).png>)
