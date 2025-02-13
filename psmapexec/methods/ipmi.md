# IPMI

IPMI is now supported. This method will attempt to dump hashes to vulnerable IPMI servers. By default, a built in user list is used unless specified in which case a user list can be queried from the domain

Successful hash output is written to `$PWD\PME\IPMI`



### Optional Parameters

<table><thead><tr><th width="158">Parameter</th><th width="169">Value</th><th>Description</th></tr></thead><tbody><tr><td>-Domain</td><td>Domain</td><td>Target domain to grab targets or a user list from</td></tr><tr><td>-SuccessOnly</td><td>N/A</td><td>Shows only successful attempts to console</td></tr><tr><td>-Option</td><td>IPMI:DomainUsers</td><td>Uses domain users as a user list against IPMI targets</td></tr><tr><td>-Option</td><td>IPMI:admin</td><td>Specify a single username to try against IPMI targets</td></tr></tbody></table>

Standard targeting user the built in user list

<pre class="language-powershell"><code class="lang-powershell"><strong>PsMapExec -Targets [Targets] -Method IPMI
</strong></code></pre>

<figure><img src="../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

Using a list of domain users as a user list, targeting all domain joined systems

```powershell
PsMapExec -Targets All -Method IPMI -Option IPMI:DomainUsers
```

<figure><img src="../../.gitbook/assets/image (2123).png" alt=""><figcaption></figcaption></figure>
