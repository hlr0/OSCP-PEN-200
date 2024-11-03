---
description: https://app.hackthebox.com/machines/list/todo
---

# Knife

## Nmap

```
sudo nmap 10.10.10.242 -p- -sS -sV

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Port 80 being the only realistic path ahead we view the root page.

![](<../../../.gitbook/assets/image (2066) (1) (1) (1) (2).png>)

We find the page is largely unusable. Running `nikto` against the target web server we see very little useful information.

```
nikto -h http://10.10.10.242
```

![](<../../../.gitbook/assets/image (265).png>)

Running `feroxbuster` against that host we see no interesting found files or directories. However, we are running PHP as per `/index.php.`

![](<../../../.gitbook/assets/image (9) (2) (1).png>)

Running the web server against ZAP we see HTTP headers return the web server running on `PHP/8.1.0-dev`.

![](<../../../.gitbook/assets/image (983).png>)

A quick Google search for this version of `PHP` immediately shows exploit code for this version.

![](<../../../.gitbook/assets/image (515).png>)

From here the first link for exploit-db.com shows us some exploit code and references for the exploit.

**Exploit-DB:** [https://www.exploit-db.com/exploits/49933](https://www.exploit-db.com/exploits/49933)

**Blog:** [https://flast101.github.io/php-8.1.0-dev-backdoor-rce/](https://flast101.github.io/php-8.1.0-dev-backdoor-rce/)

**Description**

An early release of PHP, the PHP 8.1.0-dev version was released with a backdoor on March 28th 2021, but the backdoor was quickly discovered and removed. If this version of PHP runs on a server, an attacker can execute arbitrary code by sending the User-Agent header. The following exploit uses the backdoor to provide a pseudo shell on the host.

Running the exploit and the address of the server when prompted gives us a shell as the user james.

![](<../../../.gitbook/assets/image (585).png>)

Next we notice that in /home/james/.ssh we have id\_rsa files available. We can create an authorized keys file and echo the contents of id\_rsa.pub into the authorized keys file. This will give SSH access without having any knowledge of the user james' password.

```
# Create authorized_keys file
touch authorized_keys

# echo contnets of id_rsa.pub into authorized_keys
echo "<Contents of id_rsa.pub>" > authroized_keys

# Next, copy id_rsa to the attacking system and SSH in as the strapi user.
ssh -i id_rsa james@10.10.10.242
```

![](<../../../.gitbook/assets/image (138).png>)

Once in by SSH we check `sudo -l` for sudo permissions.

![](<../../../.gitbook/assets/image (2047) (1) (1) (2).png>)

We see we can run the knife binary as the user root without specifying a password. Looking at GTFOBins we see this can be used with `sudo` to escalate privileges.

**GTFOBins:** [https://gtfobins.github.io/gtfobins/knife/](https://gtfobins.github.io/gtfobins/knife/)

![](<../../../.gitbook/assets/image (883).png>)

```
sudo /usr/bin/knife exec -E 'exec "/bin/sh"'
```

![](<../../../.gitbook/assets/image (85).png>)
