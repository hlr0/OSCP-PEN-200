# Internal

## Nmap

```
sudo nmap 10.10.185.115 -p- -sS -sV         

Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 
(Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Running dirsearch on Port 80 shows we have some interesting directories.

![](<../../../.gitbook/assets/image (881).png>)

the directory phpmyadmin shows no opportunity to login with the root account. Likely only localhost logins are permitted.

![http://10.10.185.115/phpmyadmin/index.php](<../../../.gitbook/assets/image (882).png>)

Looking back at the dirsearch results the directory /blog appears to be running wordpress as a subdirectrory was discovered of /blog/wp-admin/.

Further inspection of /blog/ shows we are indeed running Wordpress.

![http://internal.thm/blog/](<../../../.gitbook/assets/image (883) (1).png>)

Running the following wpscan command shows we have a user called admin which is soon bruteforced using the rockyou.txt wordlist.

```
 wpscan --url http://10.10.185.115/blog/ -t 40 -e at,ap,u1-3000,m1-2000 --passwords /usr/share/wordlists/rockyou.txt  
```

![](<../../../.gitbook/assets/image (884).png>)

Heading over to [http://internal.thm/blog/wp-admin/](http://internal.thm/blog/wp-admin/) we can then login with the credentials `admin:my2boys`.

Heading over to Themes and then theme editor allows us to pick a page to edit. In this example I replaced the code in the 404.php file with that of a reverse shell.

![http://internal.thm/blog/wp-admin/theme-editor.php?file=404.php\&theme=twentyseventeen](<../../../.gitbook/assets/image (885) (1).png>)

At this point usually all we need to do is browse to the 404.php page to prompt for a call back on `netcat.` This site is set up a bit different and after clicking around and following some links the directory structure for this site appears to be http://internal.thm/blog/index.php/.

I then browsed to [http://internal.thm/blog/index.php/0000/0/](http://internal.thm/blog/index.php/0000/0/) to try and get a 404.php page redirect which worked and I was able to get a shell on the system.

![](<../../../.gitbook/assets/image (886).png>)

From here I found multiple potential credentials on the machine for databases and such. None of these lead me to any escalation until I stumbled upon the /opt/ directory which contains wp-save.txt

![](<../../../.gitbook/assets/image (887).png>)

With the credentials of `aubreanna:bubb13guM!@#123` we can switch to this user. Running cat on the jenkins.txt file reveals the following information:

```
Internal Jenkins service is running on 172.17.0.2:8080
```

From our attacking host we can run the following SSH command to portforward port 8080 over to us on 127.0.0.1.

```
ssh -L 8080:172.17.0.2:8080 aubreanna@10.10.185.115
```

We can then in a web browser head over to [http://127.0.0.1:8080](http://127.0.0.1:8080)

![http://127.0.0.1:8080](<../../../.gitbook/assets/image (888).png>)

I treid known credentials against this and was unsuccessful in logging in. Bruteforcing with ZAP shows a differential response size header on the payload spongebob.

![](<../../../.gitbook/assets/image (889).png>)

Following the Groovy scipt code here change the cmd string to /bin/bash and the host to your attacking machine VPN interface IP.

[https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76](https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76)

![](<../../../.gitbook/assets/image (890).png>)

Checking our `netcat` shell after confirms we are connected as the Jenkins service.

![](<../../../.gitbook/assets/image (891) (1).png>)

Again we have information in the /opt/ directory. This time regarding SSH credentials for the root account.

![](<../../../.gitbook/assets/image (892).png>)

Which are confirmed working for a SSH connection.

![](<../../../.gitbook/assets/image (893).png>)
