# eKeys

Executes Mimikatz's sekurlsa::ekeys on each target system to retrieve Kerberos encryption keys.

For each system output is stored in `$pwd\PME\eKeys\`

**Supported Methods**

* MSSQL&#x20;
* SMB&#x20;
* SessionHunter (WMI)
* WMI&#x20;
* WinRM

### Optional Parameters

<table><thead><tr><th width="214">Parameter</th><th width="122.33333333333331">Value</th><th>Description</th></tr></thead><tbody><tr><td>-NoParse</td><td>N/A</td><td>If specified, PsMapexec will not automatically parse output from all targets systems and identify accounts that belong to privileged groups. </td></tr><tr><td>-ShowOutput</td><td>N/A</td><td>Displays each targets output to the console</td></tr><tr><td>-SuccessOnly</td><td>N/A</td><td>Display only successful results</td></tr></tbody></table>

### Usage

{% code overflow="wrap" %}
```powershell
# Standard execution
PsMapExec -Username [User] -Password [Pass] -targets [All] -Module eKeys -Method [Method] -ShowOutput
```
{% endcode %}

<figure><img src="../../.gitbook/assets/image (2) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>



### Parsing

If `-NoParse` is not specified,, PsMapExec will parse the results from each system and present the results in a digestable and readable format. The notes field will highlight in yellow any interesting information about each result. &#x20;

The table below shows the possible values for the notes field.

<table><thead><tr><th width="286">Value</th><th>Description</th></tr></thead><tbody><tr><td>AdminCount=1</td><td>The parsed account has an AdminCount value of 1. This means the account may hold some sort of privileged access within the domain.</td></tr><tr><td>rc4_hmac_nt=Empty Password</td><td>The rc4 value is equal to that of an empty password. </td></tr><tr><td>Cleartext Password</td><td>Cleartext password was parsed from the results. This is only highlited on user accounts and omitted for computer accounts.</td></tr><tr><td>Domain Admin<br>Enterprise Admin<br>Server Operator<br>Account Operator</td><td>The account is a member of a high value group.</td></tr></tbody></table>

<figure><img src="../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>









f





