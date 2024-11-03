---
description: https://attack.mitre.org/techniques/T1556/002/
---

# Password Filter DLL

**ATT\&CK ID:** [**T1556.002**](https://attack.mitre.org/techniques/T1556/002/)\*\*\*\*

\*\*Permissions Required: <mark style="color:red;">**Administrator**</mark> | \*\*<mark style="color:red;">**SYSTEM**</mark>

**Description**

Adversaries may register malicious password filter dynamic link libraries (DLLs) into the authentication process to acquire user credentials as they are validated.

Windows password filters are password policy enforcement mechanisms for both domain and local accounts. Filters are implemented as DLLs containing a method to validate potential passwords against password policies. Filter DLLs can be positioned on local computers for local accounts and/or domain controllers for domain accounts. Before registering new passwords in the Security Accounts Manager (SAM), the Local Security Authority (LSA) requests validation from each registered filter. Any potential changes cannot take effect until every registered filter acknowledges validation.

Adversaries can register malicious password filters to harvest credentials from local computers and/or entire domains. To perform proper validation, filters must receive plain-text credentials from the LSA. A malicious password filter would receive these plain-text credentials every time a password request is made.

## **Techniques - Exploit**

***

## **Mitigation**

* Ensure only valid password filters are registered. Filter DLLs must be present in Windows installation directory (C:\Windows\System32\ by default) of a domain controller and/or local computer with a corresponding entry in HKEY\_LOCAL\_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa\Notification Packages.

## **Further Reading**

***
