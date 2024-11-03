# Relevant

## Nmap

```
sudo nmap 10.10.102.118 -p- -sS -sV                                           

PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
49663/tcp open  http          Microsoft IIS httpd 10.0
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 
2012; CPE: cpe:/o:microsoft:windows
```

SMB shows the share 'nt4wrksv' when authenticating with null credentials.

![](<../../../.gitbook/assets/image (872).png>)

Using `smbclient` to connect to the share shows the file passwords.txt.

![](<../../../.gitbook/assets/image (873).png>)

The contents of passwords.txt is shown below:

```
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

We can see from the title and the trailing '==' that this is likely base64 encoded. Running both of these through base64 with the `-d` switch shows the true value.

![](<../../../.gitbook/assets/image (874).png>)

```
Bob:!P@$$W0rD!123
Bill:Juw4nnaM4n420696969!$$$ 
```

Unfortunately I was unable to use the credentials in any capacity. Referring back to SMB earlier we do have write access to the share nt4wrksv. Checking the share name and the passwords.txt file against port 49663 shows we can see the passwords.txt file.

![](<../../../.gitbook/assets/image (875).png>)

With this we can use the following aspx reverse shell found here: [https://raw.githubusercontent.com/borjmz/aspx-reverse-shell/master/shell.aspx](https://raw.githubusercontent.com/borjmz/aspx-reverse-shell/master/shell.aspx).

Upload the shell to the SMB share.

![](<../../../.gitbook/assets/image (876).png>)

Set a listener and then browse to the location of http://\<IP>/nt4wrksv/shell.aspx. Shortly after we should receive a reverse shell.

![](<../../../.gitbook/assets/image (877) (1).png>)

Checking the available privilege we do have SeImpersonatePrivilege which is not surprising as this is normally given to service accounts.

![](<../../../.gitbook/assets/image (879).png>)

I tried a Juicypotato exploit here and was unsuccessful. It seemed AV was removing my exploit every time I tried to run it. Further research on Google shows a newer exploit that abuses this privilege to gain SYSTEM which is linked here: [https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/](https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/).

We can download a pre-compiled exploit here: [https://github.com/dievus/printspoofer/raw/master/PrintSpoofer.exe](https://github.com/dievus/printspoofer/raw/master/PrintSpoofer.exe)

Once uploaded to the machine through SMB we can run with the following syntax to gain a SYSTEM shell.

```
PrintSpoofer.exe -i -c cmd
```

![](<../../../.gitbook/assets/image (880).png>)
