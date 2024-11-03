---
description: https://attack.mitre.org/techniques/T1555/003/
---

# Credentials from Web Browsers

**ATT\&CK ID:** [T1555.003](https://attack.mitre.org/techniques/T1555/003/)

**Permissions Required:** <mark style="color:green;">**User**</mark>

**Description**

\
Adversaries may acquire credentials from web browsers by reading files specific to the target browser.[\[1\]](https://blog.talosintelligence.com/2018/02/olympic-destroyer.html) Web browsers commonly save credentials such as website usernames and passwords so that they do not need to be entered manually in the future. Web browsers typically store the credentials in an encrypted format within a credential store; however, methods exist to extract plaintext credentials from web browsers.

For example, on Windows systems, encrypted credentials may be obtained from Google Chrome by reading a database file, `AppData\Local\Google\Chrome\User Data\Default\Login Data` and executing a SQL query: `SELECT action_url, username_value, password_value FROM logins;`. The plaintext password can then be obtained by passing the encrypted credentials to the Windows API function `CryptUnprotectData`, which uses the victim’s cached logon credentials as the decryption key.[\[2\]](https://docs.microsoft.com/en-us/windows/desktop/api/dpapi/nf-dpapi-cryptunprotectdata)

Adversaries have executed similar procedures for common web browsers such as FireFox, Safari, Edge, etc.[\[3\]](https://www.proofpoint.com/us/threat-insight/post/new-vega-stealer-shines-brightly-targeted-campaign)[\[4\]](https://www.fireeye.com/blog/threat-research/2017/07/hawkeye-malware-distributed-in-phishing-campaign.html) Windows stores Internet Explorer and Microsoft Edge credentials in Credential Lockers managed by the [Windows Credential Manager](https://attack.mitre.org/techniques/T1555/004).

Adversaries may also acquire credentials by searching web browser process memory for patterns that commonly match credentials.[\[5\]](https://github.com/putterpanda/mimikittenz)

## Techniques

## Firefox

### Firefox Profile Locations

```bash
# Linux
/home/<Username>/.mozilla/firefox/xxxx.default

# MacOS
/Users/<Username>/Library/Application\ Support/Firefox/Profiles/xxxx.default'

# Windows
C:\Users\<Username>\AppData\Roaming\Mozilla\Firefox\Profiles\xxxx.default
```

### Firefox\_decrypt

**Github:** [https://github.com/unode/firefox\_decrypt](https://github.com/unode/firefox\_decrypt)

```bash
python3 firefox_decrypt.py <ProfileFolder>
```

![](<../../../../.gitbook/assets/image (350).png>)

### LaZagne

```
laZagne.exe browsers browsers
```

## Google Chrome

### Chrome Profile Location

```bash
# Linux
/home/<Username>/.config/google-chrome/default

# MacOS
Users/<Username>/Library/Application Support/Google/Chrome/Default

# Windows
C:\Users\<Username>\AppData\Local\Google\Chrome\User Data\Default
```

### LaZagne

```
laZagne.exe browsers browsers
```

### Metasploit

```
use post/windows/gather/enum_chrome
```

![](<../../../../.gitbook/assets/image (1153).png>)

After completion, open the decrypted data file as mentioned by `Metasploit`.

![](<../../../../.gitbook/assets/image (1386).png>)

## Mitigation

### Disable password manager in multiple browsers

[Dashlane](https://www.dashlane.com/) have an excellent article on methods to disable the password manager for multiple browsers via Group Policy.

{% embed url="https://support.dashlane.com/hc/en-us/articles/360012461279-Disable-Chrome-Edge-Firefox-IE-password-managers-via-GPO" %}

### &#x20;<a href="#ie" id="ie"></a>
