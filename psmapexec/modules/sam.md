# SAM

Dumps SAM credentials for each target system using a heavily modified version of Invoke-[NTLMExtract.ps1](https://raw.githubusercontent.com/BC-SECURITY/Empire/main/empire/server/data/module\_source/credentials/Invoke-NTLMExtract.ps1).

For each system output is stored in $pwd\PME\PME\SAM\\

### **Supported Methods**

* MSSQL&#x20;
* SMB&#x20;
* SessionHunter (WMI)
* WMI&#x20;
* WinRM

### **Optional Parameters**

<table><thead><tr><th width="170">Parameter</th><th width="131.33333333333331">Value</th><th>Description</th></tr></thead><tbody><tr><td>-NoParse</td><td>N/A</td><td>Will ommit parsing output from each system and checks for which SAM hashes are valid on multiple systems.</td></tr><tr><td>-Rainbow</td><td>N/A</td><td>When provided, collected SAM hashes will be compared against an online database https://ntlm.pw</td></tr><tr><td>-ShowOutput</td><td>N/A</td><td>Displays each targets output to the console</td></tr><tr><td>-SuccessOnly</td><td>N/A</td><td>Display only successful results</td></tr></tbody></table>

### Usage

{% code overflow="wrap" %}
```powershell
# Standard execution
PsMapExec -Username [User] -Password [Pass] -targets [All] -Module SAM -Method [Method] -ShowOutput
```
{% endcode %}

<figure><img src="../../.gitbook/assets/image (7) (1).png" alt=""><figcaption></figcaption></figure>

### Parsing

If `-NoParse` is not specified, PsMapExec will parse the results from each system and present the results in a digestable and readable format. PsMapExec will display which systems are reusing SAM hashes and then display all collected hashes in a Hashcat friendly format.

<figure><img src="../../.gitbook/assets/image (5) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

The output appends the system name from which the hash has been pulled from to the name for easy identification. Even in this format, it is still a Hashcat friendly format.

<figure><img src="../../.gitbook/assets/image (6) (1) (1).png" alt=""><figcaption></figcaption></figure>



