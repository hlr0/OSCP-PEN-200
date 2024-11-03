---
description: https://attack.mitre.org/techniques/T1555/
---

# Credentials from Password Stores

**ATT\&CK ID:** [T1555](https://attack.mitre.org/techniques/T1555/)

**Description**

Adversaries may search for common password storage locations to obtain user credentials. Passwords are stored in several places on a system, depending on the operating system or application holding the credentials. There are also specific applications that store passwords to make it easier for users manage and maintain. Once credentials are obtained, they can be used to perform lateral movement and access restricted information.

## Sub Techniques

### T1555.003: Credentials from Web Browsers

{% content-ref url="credentials-from-web-browsers.md" %}
[credentials-from-web-browsers.md](credentials-from-web-browsers.md)
{% endcontent-ref %}

### T1555.004: Windows Credential Manager

{% content-ref url="windows-credential-manager.md" %}
[windows-credential-manager.md](windows-credential-manager.md)
{% endcontent-ref %}
