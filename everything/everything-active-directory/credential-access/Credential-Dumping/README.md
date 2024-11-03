---
description: https://attack.mitre.org/techniques/T1003/
---

# Credential Dumping

**ATT\&CK ID:** [T1003](https://attack.mitre.org/techniques/T1003/)

**Description**

Adversaries could attempt to extract credentials and account hashes from various areas of the Operating System. Clear-text passwords and hashes can be used by adversaries to perform [Lateral Movement](https://attack.mitre.org/tactics/TA0008) in the environment.

## Sub Techniques

### T1003.001: LSASS Memory

{% content-ref url="lsass-memory.md" %}
[lsass-memory.md](lsass-memory.md)
{% endcontent-ref %}

### T1003.002: Security Account Manager (SAM)

{% content-ref url="security-account-manager-sam.md" %}
[security-account-manager-sam.md](security-account-manager-sam.md)
{% endcontent-ref %}

### T1003.003: NTDS

{% content-ref url="ntds.md" %}
[ntds.md](ntds.md)
{% endcontent-ref %}

### T1003.004: LSA Secrets

{% content-ref url="lsa-secrets.md" %}
[lsa-secrets.md](lsa-secrets.md)
{% endcontent-ref %}

### T1003.005: Cached Domain Credentials

{% content-ref url="cached-domain-credentials.md" %}
[cached-domain-credentials.md](cached-domain-credentials.md)
{% endcontent-ref %}

### T1003.006: DCSync

{% content-ref url="dcsync/" %}
[dcsync](dcsync/)
{% endcontent-ref %}

