# Enumeration - Certificate Authority

## Windows

### Native&#x20;

{% tabs %}
{% tab title="AD Module" %}
{% code overflow="wrap" %}
```powershell
# AD Module
Get-ADObject -Filter * -SearchBase 'CN=Certification Authorities,CN=Public Key Services,CN=Services,CN=Configuration,DC=security,DC=local'

Get-ADObject -LDAPFilter '(objectclass=certificationAuthority)' -SearchBase 'CN=Configuration,DC=security,DC=local' | fl *
```
{% endcode %}
{% endtab %}

{% tab title="ADSI" %}
```powershell
# Get-CertificationAuthority -SearchBase LDAP://CN=Configuration,DC=security,DC=local

function Get-CertificationAuthority {
    param([string]$searchBase = "LDAP://CN=Configuration,DC=security,DC=local")
    
    $directorySearcher = New-Object System.DirectoryServices.DirectorySearcher
    $directorySearcher.SearchRoot = New-Object System.DirectoryServices.DirectoryEntry($searchBase)
    $directorySearcher.Filter = "(objectclass=certificationAuthority)"
    $directorySearcher.PropertiesToLoad.Add("*") > $null
    
    try {
        $results = $directorySearcher.FindAll()
        foreach ($result in $results) {
            $properties = @{}
            foreach ($prop in $result.Properties.PropertyNames) {
                $properties[$prop] = $result.Properties[$prop][0]
            }
            
            $outputObj = New-Object PSObject -Property $properties
            Write-Output $outputObj
        }
    }
    catch {}
    finally {
        $results.Dispose()
    }
}
```
{% endtab %}
{% endtabs %}

### Certify

Github: [https://github.com/GhostPack/Certify](https://github.com/GhostPack/Certify)

```powershell
Certify.exe cas 
Invoke-Certify cas
```

<figure><img src="../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

## Linux

### Certipy

Github: [https://github.com/ly4k/Certipy](https://github.com/ly4k/Certipy)

```python
certipy find -u <user> -p <password> -dc-ip 10.10.10.100 -stdout
```

<figure><img src="../../../.gitbook/assets/image (2) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>
