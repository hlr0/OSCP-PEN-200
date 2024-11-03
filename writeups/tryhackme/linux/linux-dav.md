---
description: https://tryhackme.com/room/bsidesgtdav
---

# Dav

## Nmap Scan

```
nmap -A -p- -T4 10.10.70.148

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
```

Looks like we only have port 80 open and looking at the `nmap` results the root page redirects to the default Apache install:

![Apache default page](<../../../.gitbook/assets/image (55) (1).png>)

Lets move onto directory enumeration.

## Dirb

Lets run a default `Dirb` scan and see what we find..

```
---- Scanning URL: http://10.10.70.148/ ----
+ http://10.10.70.148/index.html (CODE:200|SIZE:11321)
+ http://10.10.70.148/server-status (CODE:403|SIZE:300)
+ http://10.10.70.148/webdav (CODE:401|SIZE:459) 
```

Looks like we have a few hits to explore. In the meantime lets run `Gobuster`with a medium wordlist incase it picks anything up whilst we explore the above directories.

![Running Gobuster for further enumeration](<../../../.gitbook/assets/image (56) (1).png>)

## Finding credentials

`Gobuster` did not return anything of value so our next best step is to take a look at the `/webdav/` directory.

![Http-basic-auth on /webdav/ directory.](<../../../.gitbook/assets/image (57) (1).png>)

We hit a http-basic-auth login form on the directory. I tried some really basic credentials such as _admin:admin_ and\_ root:root\_ and these submissions did not work. Looks like this form is our best next step as nothing else is coming up with anything.

Initially I searched for "webdav default credentials" and "webdav ubuntu credentials" I did not find any valid information so I got a little bit more specific with "apache webdav default credentials"

![](<../../../.gitbook/assets/image (59) (1).png>)

Head over to following link and read through the blog post and you will find some valid credentials.

{% embed url="https://xforeveryman.blogspot.com/2012/01/helper-webdav-xampp-173-default.html" %}

After obtaining some valid credentials we are able to login to the server.

![/webdav/ directory post authorization](<../../../.gitbook/assets/image (60) (1).png>)

Inside the passwd.dav file is a username and a hash. I run the hash through `John The Ripper` on this and the value returned was the same password we used to authenticate with. I ran `Dirb` on the authenticated directory with the username:password format added to the command and this returned nothing.

With no other clear avenue of attack I decided to test using a HTTP PUT request on the directory. I started `Burpsuite` and added the IP address of the server to the Scope.

![Adding the server IP to Scope](<../../../.gitbook/assets/image (62) (1).png>)

Ensure intercept is turned on and refresh the `/webdav/` page in the browser. You should receive the following which we will need to send through to repeater.

![Burp has intercepted our GET request for the /webdav/ directory.](<../../../.gitbook/assets/image (63) (1).png>)

{% hint style="info" %}
To send the following through to repeater right click anywhere in the Raw tab and hit "Send to repeater"
{% endhint %}

We now need to change the GET request to a PUT request. All we need to do is change `GET` to `PUT` and add the file name and extension after `/webdav/`

![Creating our PUT request](<../../../.gitbook/assets/image (64) (1).png>)

Below this we need to paste our payload code. In this instance we will be using a PHP reverse shell from [pentestmoneky.net](http://pentestmonkey.net)

Here we change the IP to our VPN interface and set the port. In this instance I have set port 443 as it is usually reliable for reverse shells.

![Adding the PHP code from a reverse shell](<../../../.gitbook/assets/image (65) (1).png>)

All we need to go now is select "Go" and see what the response on the right says.

![HTTP response for a PUT request](<../../../.gitbook/assets/image (66) (1).png>)

As we can see the HTTP response informs us the /webdav/shell.php resource has been created. Turn intercept off and refresh the directory and you should see the shell has bee uploaded.

![](<../../../.gitbook/assets/image (67) (1).png>)

Before we run the shell lets start a `netcat` listener for the specified port. In this case 443. We will need to use sudo if we are using a port from 1-1024

![Starting a listener with Netcat](<../../../.gitbook/assets/image (68) (1).png>)

We can now click and on the shell.php and we should receive a reverse shell as a low privilege user.

![](<../../../.gitbook/assets/image (70).png>)

First things first lets upgrade our shell to a python shell using the following:

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

I checked the html directory and found nothing interesting. After checking home we have two users "wampp" and "merlin". We have access to merlin's home directory and from here we can find the user.txt flag and as you can see we have read permissions to the file. Grab the flag before moving on.

![](<../../../.gitbook/assets/image (71) (1) (1).png>)

## Reading Root

Let's start of by starting a python SimpleHTTPServer on our attacking machine so we can transfer Linpeas over. `cd` over to the directory containing Linpeas and run the following code:

```
python -m SimpleHTTPServer
```

Then we can run `wget` on the victim machine to download the Linpeas file and from here we can execute Linpeas and pipe it to tee so we can read the results when it is finished.

![Running wget against the IP of our attacking machine](<../../../.gitbook/assets/image (72) (1).png>)

An interesting piece of information has been found by Linpeas:

![Linpeas running 'sudo -l'](<../../../.gitbook/assets/image (73) (1).png>)

Since we can run cat as sudo we could just simply run the following command to read the root flag:

![](<../../../.gitbook/assets/image (74) (1).png>)

This box is now complete.
