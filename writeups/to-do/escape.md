# Escape (WIP)

## Nmap

```
sudo nmap 192.168.233.113 -p- -sS -sV

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
8080/tcp open  http    Apache httpd 2.4.38 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

![](<../../.gitbook/assets/image (980) (1).png>)

The directory page /dev/index.php allows for GIF uploads. Attempting to upload a different file type will be rejected.

![](<../../.gitbook/assets/image (981).png>)

This can be bypassed by first upload a legitimate GIF file. I used a small GIF such as the following[: https://giphy.com/gifs/obi-won-hvE0PhVAnGQAo](https://giphy.com/gifs/obi-won-hvE0PhVAnGQAo).

Upload the file and capture the request with Burpsuite. The request when attempting upload should look something similar to the image below:

![](<../../.gitbook/assets/image (982).png>)

We now need to send the request to repeater so we can edit it and inject a PHP reverse shell. In the request below I have ensured the filename ends in '.php' and the first line of the original request has been kept in as these are magic bytes which are used to identify what type of file is being upload. As far as the server is concerned because the line starting with: `GIF89aÈ*` exists in the request then the file must be a GIF.

Ensure the following is still set: `Content-Type: image/gif` Then under the magic byte sequence we can inject a PHP Reverse shell.

![](<../../.gitbook/assets/image (983) (1).png>)

Once completed send the request on and check to see if uploaded.

![](<../../.gitbook/assets/image (984).png>)

Once uploaded start a netcat listener and browse to the following:[http://192.168.233.113:8080/dev/uploads/shell.php](http://192.168.233.113:8080/dev/uploads/shell.php).

A shell should be received as www-data.

![](<../../.gitbook/assets/image (985).png>)
