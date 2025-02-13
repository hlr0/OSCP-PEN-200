# NTDS

Executes Mimikatz's lsadump::dcsync on the target system. Parses the NTDS file to replicate Secretsdump output. No files are created on disk on the target system.

Output for each system is stored in $pwd\PME\NTDS\\

### **Supported Methods**

* MSSQL&#x20;
* SMB&#x20;
* SessionHunter (WMI)
* WMI&#x20;
* WinRM

### **Optional Parameters**

<table><thead><tr><th width="170">Parameter</th><th width="94.33333333333331">Value</th><th>Description</th></tr></thead><tbody><tr><td>-NoParse</td><td>N/A</td><td>Will ommit parsing output from the method. Will Simply extract the NTDS file in a hashcat friendly format</td></tr><tr><td>-Rainbow</td><td>N/A</td><td>When provided, collected hashes will be compared against an online database https://ntlm.pw</td></tr><tr><td>-ShowOutput</td><td>N/A</td><td>Displays each targets output to the console</td></tr><tr><td>-SuccessOnly</td><td>N/A</td><td>Display only successful results</td></tr></tbody></table>

### Usage

{% code overflow="wrap" %}
```powershell
# Standard execution
PsMapExec -Username [User] -Password [Pass] -targets [DC] -Module NTDS -Method [Method] -ShowOutput
```
{% endcode %}

<figure><img src="../../.gitbook/assets/image (3) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

### Parsing

If `-NoParse` is not specified, PsMapExec will parse the results from the NTDS output and present them in a digestable and usable format.

<figure><img src="../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>
