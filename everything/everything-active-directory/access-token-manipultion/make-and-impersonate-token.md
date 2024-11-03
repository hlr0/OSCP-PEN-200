---
description: https://attack.mitre.org/techniques/T1134/003/
---

# 🔨 Make and Impersonate Token

**ATT\&CK ID:** [T1134.003](https://attack.mitre.org/techniques/T1134/003/)

**Permissions Required:** <mark style="color:red;">**Administrator**</mark> | <mark style="color:green;">**User**</mark>

**Description**

Adversaries may make and impersonate tokens to escalate privileges and bypass access controls. If an adversary has a username and password but the user is not logged onto the system, the adversary can then create a logon session for the user using the `LogonUser` function. The function will return a copy of the new session's access token and the adversary can use `SetThreadToken` to assign the token to a thread.

[\[Source\]](https://attack.mitre.org/techniques/T1134/003/)

## Techniques
