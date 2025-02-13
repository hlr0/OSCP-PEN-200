# KerbDump

Runs Kirby to dump kerberos tickets on the remote system. Kirby is based on PowerShellKerberos by Michael Zhmaylo (MzHmO): [https://github.com/MzHmO/PowershellKerberos](https://github.com/MzHmO/PowershellKerberos)

For each system output is stored in $pwd\PME\Tickets\KerbDump\\

### **Supported Methods**

* MSSQL&#x20;
* SMB&#x20;
* SessionHunter (WMI)
* WMI&#x20;
* WinRM

### Optional Parameters

<table data-full-width="false"><thead><tr><th width="180">Parameter</th><th width="117.33333333333331">Value</th><th>Description</th></tr></thead><tbody><tr><td>-NoParse</td><td>N/A</td><td>If specified, PsMapexec will not automatically parse output from all targets systems and identify accounts that belong to privileged groups.</td></tr><tr><td>-ShowOutput</td><td>N/A</td><td>Displays each targets output to the console</td></tr><tr><td>-SuccessOnly</td><td>N/A</td><td>Display only successful results</td></tr></tbody></table>

### Usage

{% code overflow="wrap" %}
```powershell
# Standard execution
PsMapExec -Username [User] -Password [Pass] -targets [All] -Module KerbDump -Method [Method] -ShowOutput
```
{% endcode %}

<figure><img src="../../.gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

### Parsing

If `-NoParse` is not specified,  PsMapExec will parse the results from each system and present the results in a digestable and readable format. The notes field will highlight in yellow any interesting information about each result. &#x20;

Tickets identified as a TGT will also show an easy command to execute directly after with PsMapExec to impersonate that account within the Impersonate field.

The table below shows the possible values for the notes field.

<table><thead><tr><th width="205.33333333333331">Value</th><th>Description</th></tr></thead><tbody><tr><td>TGT</td><td>Represents a TGT ticket</td></tr><tr><td>AdminCount=1</td><td>Identifies an account that may hold privileged permissions within the domain</td></tr><tr><td>Domain Admin<br>Enterprise Admin<br>Server Operator<br>Account Operator</td><td>The account is a member of one of these privileged groups</td></tr></tbody></table>

<figure><img src="../../.gitbook/assets/image (2105).png" alt=""><figcaption></figcaption></figure>
