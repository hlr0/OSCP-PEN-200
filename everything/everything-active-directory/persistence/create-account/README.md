---
description: https://attack.mitre.org/techniques/T1136/
---

# Create Account

**ATT\&CK ID:** [T1136](https://attack.mitre.org/techniques/T1136/)

**Description**

Adversaries may create an account to maintain access to victim systems. With a sufficient level of access, creating such accounts may be used to establish secondary credentialed access that do not require persistent remote access tools to be deployed on the system.

Accounts may be created on the local system or within a domain or cloud tenant. In cloud environments, adversaries may create accounts that only have access to specific services, which can reduce the chance of detection.

## **Sub Techniques**

### **T1136.001: Local Account**

{% content-ref url="local-account.md" %}
[local-account.md](local-account.md)
{% endcontent-ref %}

### **T1136.002: Domain Account**

{% content-ref url="domain-account.md" %}
[domain-account.md](domain-account.md)
{% endcontent-ref %}

### **T1136.003: Cloud Account**

{% content-ref url="cloud-account.md" %}
[cloud-account.md](cloud-account.md)
{% endcontent-ref %}
