---
description: https://app.hackthebox.com/machines/Cap
---

# Cap

## Nmap

```
nmap 10.10.10.245 -p- -sS -sV

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    gunicorn
Nmap done: 1 IP address (1 host up) scanned in 141.34 seconds
```

Initially we start out checking FTP to see if anonymous log in is enabled which, unfortuantely is not the case.

We next move onto Port 80 and notice we are taken to a page titled Security Dashboard.

![](<../../../.gitbook/assets/image (430).png>)

Preliminary checks on the user interface show we are able to view basis `netstat` information and IP addresses assigned to local interfaces on the target system.

![](<../../../.gitbook/assets/image (2065) (1) (1) (1) (1) (2).png>)

We also notice that we can download packet capture `PCAP` files.

![](<../../../.gitbook/assets/image (1134).png>)

Every time we request a recapture under `/capture` the download link for the `PCAP` file increments by 1.

This starts from `/data/1`. When we browse to `/data/0` we are able to download an unrequested `PCAP` file.

Loading the resulting `PCAP` file into `Wireshark` we are able to view credentials for the user Nathan.

![](<../../../.gitbook/assets/image (705).png>)

Which gives us FTP credentials for `nathan:Buck3tH4TF0RM3!`

These same credentials can be used for SSH access.

```
ssh nathan@10.10.10.245
```

![](<../../../.gitbook/assets/image (381).png>)

Now logged on we begin enumerating. Checking what capabilities are enabled on the target system we see Python 3 has some interesting ones set.

```
getcap -r / 2>/dev/null
```

![](<../../../.gitbook/assets/image (17) (2).png>)

The `cap+setuid_capabilty` will allow us to call a `python` command with a uid of '0' which is root. We can use these to spawn a root bash shell.

```
python3 -c 'import os;os.setuid(0);os.system("/bin/bash")'
```

![](<../../../.gitbook/assets/image (282).png>)
