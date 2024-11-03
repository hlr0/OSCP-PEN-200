# 🔨 LLMNR

## What is it

Link-Local Multicast Name Resolution (LLMNR) is a protocol that is able to perform name resolution in the absence of a DNS server. When a request is made for a share such as `\\Fileshares` and the user accidently instead types `\\Filesharez` getting the share name incorrect DNS would first attempt to resolve the target share and when failing to do so (As it does not exist) It will fallback to LLMNR.

Once DNS has failed to resolve the request and LLMNR kicks in the requesting machine will send out a broadcast on the subnet asking if anyone of the other devices can connect them to the share `\\Filesharez` The attacking machine on the network will respond to the request stating that it can get them connected to the share. At this point the requesting (victim) machine will send the username and NTLMv2 hash of the account requesting the resource over to the malicious machine.

## Performing the attack

To perform the attack we need to first start up Responder on Kali.

{% hint style="info" %}
As of the time of writing December 2nd 2020 the version of Responder built into Kali v3.0.2.0 did not work correctly for me when attempting to capture hashes so instead I downloaded version 3.0.0.0 from Igandx's Github.
{% endhint %}

{% embed url="https://github.com/lgandx/Responder/releases" %}

Once ready we can start Responder with the following command:

```
sudo python Responder.py -I eth0 -wdr
```

Once started you should see the following:

![](<../../../.gitbook/assets/image (1518).png>)

For now, on the attacking machine this is all we need to do. We now need to wait for an event to happen in which LLMNR is triggered by the victim machine so Responder can reply back to the victim machine to poison then capture the NTLMv2 hash.

On the victim machine if we was a user and tried accessing a file share where the name is incorrect or does not exist in the DNS server you would be prompted with the following message as per below.

![](<../../../.gitbook/assets/image (1519).png>)

In the example above the user has attempted to access the share `\\filesharez` which does not exsist. Firstly the machine will refer to the DNS server and attempt to resolve this name to IP. On failure to resolve the machine will send out a broadcast on the subnet using LLMNR asking if anyone is able to get the machine connected to the share.

At this point Responder which is actively listening will respond to this request asking for the requesting machine users NTLMv2 hash so it can masquerade as passing it on to the correct resolver.

Once this happens Responder should pick up the hash as shown below:

![](<../../../.gitbook/assets/image (1520).png>)

Once the hash is captured we can take it offline and attempt to crack it using a tool such as Hashcat.

We can use the following command to crack the hash:

```
hashcat -m 5600 <hashFile> <Wordlist> --force -O
```

The switch `-m` indicates which mode of attack to use on the hash. 5600 is defined in the Hashcat examples page as being for a NTLMv2 hash. [https://hashcat.net/wiki/doku.php?id=example\_hashes](https://hashcat.net/wiki/doku.php?id=example\_hashes)

![](<../../../.gitbook/assets/image (1521).png>)

After reading the output we can see hashcat has cracked the password as 'iloveyou' We now have the complete credentials of 'Bart.Simpson:iloveyou'

### Mitigations <a href="#mitigations" id="mitigations"></a>

| Mitigation                                                                         | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Disable or Remove Feature or Program](https://attack.mitre.org/mitigations/M1042) | Disable LLMNR and NetBIOS in local computer security settings or by group policy if they are not needed within an environment. [\[14\]](https://adsecurity.org/?p=3299)                                                                                                                                                                                                                                                                                                                |
| [Filter Network Traffic](https://attack.mitre.org/mitigations/M1037)               | Use host-based security software to block LLMNR/NetBIOS traffic. Enabling SMB Signing can stop NTLMv2 relay attacks.[\[3\]](https://byt3bl33d3r.github.io/practical-guide-to-ntlm-relaying-in-2017-aka-getting-a-foothold-in-under-5-minutes.html)[\[4\]](https://blog.secureideas.com/2018/04/ever-run-a-relay-why-smb-relays-should-be-on-your-mind.html)[\[15\]](https://docs.microsoft.com/en-us/previous-versions/system-center/operations-manager-2005/cc180803\(v=technet.10\)) |
| [Network Intrusion Prevention](https://attack.mitre.org/mitigations/M1031)         | Network intrusion detection and prevention systems that can identify traffic patterns indicative of MiTM activity can be used to mitigate activity at the network level.                                                                                                                                                                                                                                                                                                               |
| [Network Segmentation](https://attack.mitre.org/mitigations/M1030)                 | Network segmentation can be used to isolate infrastructure components that do not require broad network access. This may mitigate, or at least alleviate, the scope of MiTM activity.                                                                                                                                                                                                                                                                                                  |

### Detection <a href="#detection" id="detection"></a>

Monitor `HKLM\Software\Policies\Microsoft\Windows NT\DNSClient` for changes to the "EnableMulticast" DWORD value. A value of "0" indicates LLMNR is disabled. [\[16\]](https://www.sternsecurity.com/blog/local-network-attacks-llmnr-and-nbt-ns-poisoning)

Monitor for traffic on ports UDP 5355 and UDP 137 if LLMNR/NetBIOS is disabled by security policy.

Deploy an LLMNR/NBT-NS spoofing detection tool.[\[17\]](https://github.com/Kevin-Robertson/Conveigh) Monitoring of Windows event logs for event IDs 4697 and 7045 may help in detecting successful relay techniques.[\[4\]](https://blog.secureideas.com/2018/04/ever-run-a-relay-why-smb-relays-should-be-on-your-mind.html)

**Source**: [https://attack.mitre.org/techniques/T1557/001/](https://attack.mitre.org/techniques/T1557/001/)

## Resources

{% embed url="https://www.aptive.co.uk/blog/llmnr-nbt-ns-spoofing/" %}

{% embed url="https://medium.com/@subhammisra45/llmnr-poisoning-and-relay-5477949b7bef" %}

{% embed url="https://attack.mitre.org/techniques/T1557/001/" %}
