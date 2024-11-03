# ConsoleHistory

### **Description**

Enumerates for and reads the ConsoleHost\_history.txt file within each accessible user directory. This file can often contain credentialed information that has been stored within the terminal

For each system output is stored in $pwd\PME\PME\Console History\\

### **Supported Methods**

* MSSQL&#x20;
* SMB&#x20;
* SessionHunter (WMI)
* WMI&#x20;
* WinRM

### Optional Parameters

<table><thead><tr><th width="180">Parameter</th><th width="127.33333333333331">Value</th><th>Description</th></tr></thead><tbody><tr><td>-ShowOutput</td><td>N/A</td><td>Displays each targets output to the console</td></tr><tr><td>-SuccessOnly</td><td>N/A</td><td>Display only successful results</td></tr></tbody></table>

### Usage

{% code overflow="wrap" %}
```powershell
# Standard execution
PsMapExec -Username [User] -Password [Pass] -targets [All] -Module ConsoleHistory -Method [Method] -ShowOutput
```
{% endcode %}

<figure><img src="../../.gitbook/assets/image (546).png" alt=""><figcaption></figcaption></figure>
