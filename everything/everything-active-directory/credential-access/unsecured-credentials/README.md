---
description: https://attack.mitre.org/techniques/T1552/
---

# Unsecured Credentials

**ATT\&CK ID:** [T1552](https://attack.mitre.org/techniques/T1552/)

**Description**

Adversaries may search compromised systems to find and obtain insecurely stored credentials. These credentials can be stored and/or misplaced in many locations on a system, including plaintext files (e.g. Bash History), operating system or application-specific repositories (e.g. Credentials in Registry), or other specialized files/artifacts (e.g. Private Keys).

## Sub Techniques

### T1552:001: Credentials in Files

{% content-ref url="credentials-in-files.md" %}
[credentials-in-files.md](credentials-in-files.md)
{% endcontent-ref %}

### T1552:002: Credentials in Registry

{% content-ref url="credentials-in-registry.md" %}
[credentials-in-registry.md](credentials-in-registry.md)
{% endcontent-ref %}

### T1552:006: Group Policy preferences

{% content-ref url="group-policy-preferences/" %}
[group-policy-preferences](group-policy-preferences/)
{% endcontent-ref %}
