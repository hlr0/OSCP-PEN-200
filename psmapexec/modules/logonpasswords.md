# LogonPasswords

Executes Mimikatz's sekurlsa::logonpasswords on the target system.&#x20;

Output for each system is stored in $pwd\PME\LogonPasswords\\

### **Supported Methods**

* MSSQL&#x20;
* SMB&#x20;
* SessionHunter (WMI)
* WMI&#x20;
* WinRM

### Optional Parameters

<table><thead><tr><th width="165">Parameter</th><th width="119.33333333333331">Value</th><th>Description</th></tr></thead><tbody><tr><td>-NoParse</td><td>N/A</td><td>If specified, PsMapexec will not automatically parse output from all targets systems and identify accounts that belong to privileged groups. </td></tr><tr><td>-Rainbow</td><td>N/A</td><td>When provided, collected hashes will be compared against an online database https://ntlm.pw</td></tr><tr><td>-ShowOutput</td><td>N/A</td><td>Displays each targets output to the console</td></tr><tr><td>-SuccessOnly</td><td>N/A</td><td>Display only successful results</td></tr></tbody></table>

### Usage

{% code overflow="wrap" %}
```
# Standard execution
PsMapExec -Username [User] -Password [Pass] -targets [All] -Module LogonPasswords -Method [Method] -ShowOutput
```
{% endcode %}

<figure><img src="../../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

### Parsing

If `-NoParse`  is not specified, , PsMapExec will parse the results from each system and present the results in a digestable and readable format. The notes field will highlight in yellow any interesting information about each result. &#x20;

The table below shows the possible values for the notes field.

<table><thead><tr><th width="242">Value</th><th>Description</th></tr></thead><tbody><tr><td>AdminCount=1</td><td>The parsed account has an AdminCount value of 1. This means the account may hold some sort of privileged access within the domain.</td></tr><tr><td>NTLM=Empty Password</td><td>The NTLM value is equal to that of an empty password. </td></tr><tr><td>Cleartext Password</td><td>Cleartext password was parsed from the results. This is only highlited on user accounts and omitted for computer accounts.</td></tr><tr><td>Domain Admin<br>Enterprise Admin<br>Server Operator<br>Account Operator</td><td>The account is a member of a high value group.</td></tr></tbody></table>



<figure><img src="../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

At the end of parsing all unique NTLM hashes will be shown in the console window. A Hashcat ready file will also be populated for collected NTLM hashes in:

&#x20;`$pwd\PME\LogonPasswords\.AllUniqueNTLM.txt`

<figure><img src="../../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>
