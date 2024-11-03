---
description: https://attack.mitre.org/techniques/T1070/003/
---

# Clear Command History

**ATT\&CK ID:** [T1070.003](https://attack.mitre.org/techniques/T1070/003/)

**Description**

In addition to clearing system logs, an adversary may clear the command history of a compromised account to conceal the actions undertaken during an intrusion. Various command interpreters keep track of the commands users type in their terminal so that users can retrace what they've done.

On Windows hosts, PowerShell has two different command history providers: the built-in history and the command history managed by the `PSReadLine` module. The built-in history only tracks the commands used in the current session. This command history is not available to other sessions and is deleted when the session ends.

The `PSReadLine` command history tracks the commands used in all PowerShell sessions and writes them to a file (`$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt` by default). This history file is available to all sessions and contains all past history since the file is not deleted when the session ends.

Adversaries may run the PowerShell command `Clear-History` to flush the entire command history from a current PowerShell session. This, however, will not delete/flush the `ConsoleHost_history.txt` file. Adversaries may also delete the `ConsoleHost_history.txt` file or edit its contents to hide PowerShell commands they have run.

\[[Source](https://attack.mitre.org/techniques/T1070/003/)]

## **Techniques**

**The techniques for this page are the same as tactic** [T1562.003](https://attack.mitre.org/techniques/T1562/003/) (Impair Command History Logging). The below links redirect to the techniques related:

**GitBook Page Link**

{% content-ref url="../impair-defenses/impair-command-history-logging.md" %}
[impair-command-history-logging.md](../impair-defenses/impair-command-history-logging.md)
{% endcontent-ref %}

**URL Link**

[https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/defense-evasion/impair-defenses/impair-command-history-logging](https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/defense-evasion/impair-defenses/impair-command-history-logging)

## **Mitigation**

* Making the environment variables associated with command history read only may ensure that the history is preserved.
* Forward logging of historical data to remote data store and centralized logging solution to preserve historical command line log data.

## **Further Reading**

**PowerShell History File:** [https://0xdf.gitlab.io/2018/11/08/powershell-history-file.html](https://0xdf.gitlab.io/2018/11/08/powershell-history-file.html)
