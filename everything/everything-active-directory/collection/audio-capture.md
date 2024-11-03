---
description: https://attack.mitre.org/techniques/T1123/
---

# Audio Capture

**ATT\&CK ID:** [T1123](https://attack.mitre.org/techniques/T1123/)

**Permissions Required:** <mark style="color:green;">**User**</mark>

**Description**

An adversary can leverage a computer's peripheral devices (e.g., microphones and webcams) or applications (e.g., voice and video call services) to capture audio recordings for the purpose of listening into sensitive conversations to gather information.

Malware or scripts may be used to interact with the devices through an available API provided by the operating system or an application to capture audio. Audio files may be written to disk and exfiltrated later.

\[[Source](https://attack.mitre.org/techniques/T1123/)]

## Techniques

### Metasploit

Captures audio input from the hosts primary microphone device. the parameter `-i` specifies the recording interval in seconds.

```
run sound_recorder -i 300 -l /home/kali
```

![](<../../../.gitbook/assets/image (375).png>)

## Mitigation

* Monitor executed commands and arguments for actions that can leverage a computer’s peripheral devices (e.g., microphones and webcams) or applications (e.g., voice and video call services) to capture audio recordings for the purpose of listening into sensitive conversations to gather information.
* Disconnect devices capable of audio input, muting is less preferable unless the device is governed by a physical switch.

## Further Reading

* [https://null-byte.wonderhowto.com/how-to/hack-like-pro-remotely-record-listen-microphone-anyones-computer-0143966/](https://null-byte.wonderhowto.com/how-to/hack-like-pro-remotely-record-listen-microphone-anyones-computer-0143966/)
