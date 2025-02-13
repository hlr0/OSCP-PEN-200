# ESC2

## Description

ESC2 works on the same core principal as ESC1, where a low privileged user or group has the ability to supply a subjectAltName (SAN) for any other user or machine in Active Directory. in ESC1 attacks the flags for  the Extended Key Usage (EKU) need to contain "Client Authentication" to be valid.\
\
ESC2 by comparison is where the EKU is set to "Any Purpose" or is void of any usage specifications

The attack method for this follows much the same as ESC1 except there is a small variation in the "pre-requisites"&#x20;

### Tools Required

* Certify
* Rubeus
* OpenSSL: [https://slproweb.com/download/Win64OpenSSL\_Light-3\_2\_1.msi](https://slproweb.com/download/Win64OpenSSL\_Light-3\_2\_1.msi)
* Access to a UNIX system if unable to install OpenSSL on the testing Windows host

### Requirements for attack path

* ENROLLEE\_SUPPLIES\_SUBJECT flag in the certificate template
* Enrolment rights granted to a user or group for which we have access to
* EKU is set to "Any Purpose" or "null"
* Manager approval not enabled
* Authorized signature are not required

## Enumeration

### Windows

{% tabs %}
{% tab title="Certify" %}
```powershell
Invoke-Certify find /enrolleeSuppliesSubject /enabled
```
{% endtab %}

{% tab title="AD Module" %}
{% code overflow="wrap" %}
```powershell
Get-ADObject -LDAPFilter '(&(objectclass=pkicertificatetemplate)(!(mspki-enrollment-flag:1.2.840.113556.1.4.804:=2))(|(mspki-ra-signature=0)(!(mspki-ra-signature=*)))(|(pkiextendedkeyusage=2.5.29.37.0)(!(pkiextendedkeyusage=*))))' -SearchBase 'CN=Configuration,DC=security,DC=local' | FL
```
{% endcode %}
{% endtab %}
{% endtabs %}

<figure><img src="../../../.gitbook/assets/image (3) (1) (1).png" alt=""><figcaption></figcaption></figure>

### Linux

{% code overflow="wrap" %}
```python
# Syntax
certipy find -u <User@Domain.local> -p <Password> -dc-ip <IP> -enabled -stdout

# Example
certipy find -u truth@security.local -p Password123! -dc-ip 10.10.10.100 -enabled -stdout
```
{% endcode %}

## Performing The Attack

Follow the attack process described below. The attack is the same for ESC1 after identifying a vulnerable certificate template.&#x20;

{% content-ref url="esc1.md" %}
[esc1.md](esc1.md)
{% endcontent-ref %}

## Mitigations

* Remove the ENROLEE\_SUPPLIES\_SUBJECT flag from the certificate template
* Remove the "Any Purpose" EKU from the template, EKU's should be given specific use definitions
* Ensure Manager approval is required on the certificate
* Require authorized signatures
* If possible, remove Enrollment rights for low privileges groups such as Domain users and Domain Computers
