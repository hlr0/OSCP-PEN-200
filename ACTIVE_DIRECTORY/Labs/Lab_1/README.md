# OSCP-TRAINING-Vulnerable-Active-Directory-2019-with-WIN10-client-for-PROXMOX

```
Author: h1r0
Date: 02/15/2024
```

### DESCRIPTION
Setup a vulnerable active directory 2019 with windows 10 proxmox hacklab\
Install Windows server 2019 and windows 10 like normal with normal admin users\
then start with the below to make it into a vulnerable hack lab.\
With Randomized attackes you can practice various aspects including some attackes require the client workstation

### Attack Vectors
Read the Attack Vectors Readme file...
| Attack Types Group I           | Attack Types Group II                             | Attack Types Group III   | 
| ------------                   | ------------                                      | ------------             |
| Abusing ACLs/ACEs              | Kerberoasting                                     | SMB Signing Disabled     |
| AS-REP Roasting                | Abuse DnsAdmins                                   | Pass-the-Ticket          |
| Password in Object Description | User Objects With Default password (Changeme123!) | Pass-the-Hash            |
| Password Spraying              | DCSync                                            | Golden Ticket            |
| Silver Ticket                  |                                                   |                          |
 
### Requirements
- Proxmox box
- VM Windows 2019 Server Datacenter (Desktop Experience)
- VM Windows 10 Pro

### Setup Modifications
- Change the ip address to your network configuration
- Check which network adapter your using 

# #####################################################################################
# #####################################################################################
# #####################################################################################
### WINDOWS SERVER 2019 SETUP
##### RUN THESE COMMANDS MANUAL IN POWERSHELL WITH ADMINISTRATOR ON THE DC01
# #####################################################################################

we can rename the VM and configure the network. Renaming to DC01 with PowerShell without rebooting.
```
c:\>Rename-Computer -NewName "DC01" -Force -Restart:$false
```

Setting a fixed IP address, you can retrieve the interface index with the Get-NetAdapter command
```
c:\>Get-NetAdapter
c:\>New-NetIPAddress -InterfaceIndex 7 -IPAddress "10.254.3.180" -AddressFamily IPv4 -PrefixLength 24 -DefaultGateway "10.254.3.1"
```

DNS configuration
```
c:\>Set-DnsClientServerAddress -InterfaceIndex 7 -ServerAddresses "localhost"
```

Renaming the Ethernet interface to lan.
```
c:\>Rename-NetAdapter -Name "Ethernet 2" -NewName lan
```

Check the configuration effects
```
c:\>ipconfig /all
```

### REBOOT THE SYSTEM

install the AD DS role with the following command
```
c:\>Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

create our first forest and promote DC01 as a domain controller.
```
c:\>Install-ADDSForest -DomainName "lab.int" -DomainNetBiosName "LAB" -InstallDns:$true -NoRebootOnCompletion:$true
```

Enter the DSRM password twice, which is a secure mode boot method for Windows Server DCs, then press Y to confirm the configuration.

### REBOOT THE SYSTEM

Create a lskywalker user on the DC

```
c:\>New-ADUser -Name "Luke Skywalker" -GivenName "Luke" -Surname "Skywalker" -SamAccountName "lskywalker" -UserPrincipalName "lskywalker@lab.int" -AccountPassword (ConvertTo-SecureString -AsPlainText "P@ssword" -Force) -Enabled $true -ChangePasswordAtLogon $true -Path "CN=Users,DC=lab,DC=int"
```
### REBOOT THE SYSTEM

# #####################################################################################
# #####################################################################################
# #####################################################################################
### WINDOWS 10 BOX COMMANDS SETUP
##### RUN THESE COMMANDS MANUAL IN POWERSHELL WITH ADMINISTRATOR ON THE CLIENT COMPUTER 
# #####################################################################################

Powershell as administrator, rename host
```
c:\>Rename-Computer -NewName "CPC01" -Force -Restart:$false
```

Setting a fixed IP address
```
c:\>Get-NetAdapter
c:\>New-NetIPAddress -InterfaceIndex 2 -IPAddress "10.254.3.185" -AddressFamily IPv4 -PrefixLength 24 -DefaultGateway "10.254.3.1"
```

DNS Configuration
```
c:\>Set-DnsClientServerAddress -InterfaceIndex 2 -ServerAddresses "10.254.3.180"
```
Renaming the Ethernet interface to lan.
```
c:\>Rename-NetAdapter -Name "Ethernet" -NewName lan
```

Check the configuration effects
```
c:\>ipconfig /all
```

### REBOOT

Add the client to the domain, from the client computer in powershell (admin)
```
c:\>Add-Computer -DomainName lab.int -Credential Administrator@lab.int
```

### REBOOT

# #####################################################################################
# #####################################################################################
# #####################################################################################
# #####################################################################################
### WINDOWS SERVER 2019 SETUP
##### SCRIPT TO MAKE THE ACTIVE DIRECTORY VULNERABLE
# #####################################################################################

Create a vulnerable active directory that's allowing you to test most of active directory attacks in local lab
Main Features

    Randomize Attacks
    Full Coverage of the mentioned attacks
    you need run the script in DC with Active Directory installed
    Some of attacks require client workstation

Supported Attacks

    Abusing ACLs/ACEs
    Kerberoasting
    AS-REP Roasting
    Abuse DnsAdmins
    Password in Object Description
    User Objects With Default password (Changeme123!)
    Password Spraying
    DCSync
    Silver Ticket
    Golden Ticket
    Pass-the-Hash
    Pass-the-Ticket
    SMB Signing Disabled

### Setup

Unrestricted policy loads all configuration files and runs all scripts. If you run an unsigned script that was downloaded from the Internet, you are prompted for permission before it runs.
Whereas in Bypass policy, nothing is blocked and there are no warnings or prompts during script execution. Bypass ExecutionPolicy is more relaxed than Unrestricted.

```
c:\>Get-ExecutionPolicy

c:\>Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy Bypass -Force;

#Use this one rather
c:\>Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy Unrestricted -Force;
```

Run the vulnad.ps1 script from this repo in powershell
```
#be mindful of the double dot
c:\>. .\vulnad.ps1
c:\>Invoke-VulnAD -UsersLimit 100 -DomainName "lab.int"
```

After a few minutes, the environment is configured.
![output1](https://raw.githubusercontent.com/hlr0/OSCP-Vulnerable-AD2019-WIN10-proxmox/main/img/vulnad.png)
![output2](https://raw.githubusercontent.com/hlr0/OSCP-Vulnerable-AD2019-WIN10-proxmox/main/img/badacl.png)

In the Active Directory Users And Computers console, there are multiple users, multiple groups and vulnerability settings.

![usersoutput](https://raw.githubusercontent.com/hlr0/OSCP-Vulnerable-AD2019-WIN10-proxmox/main/img/users.png)
![groupoutput](https://raw.githubusercontent.com/hlr0/OSCP-Vulnerable-AD2019-WIN10-proxmox/main/img/groups.png)

# #####################################################################################
### WINDOWS 10 BOX
##### CHECK THAT THIS IS ALL WORKING
# #####################################################################################

Use the "lskywalker" user with password of "P@ssword" to login with the client machine\n
You will be required to change the password from the client machine... 

# DONE...HAVE FUN -- & ENJOY HACKING THIS UP...!!!
# REMEMBER -- TRY HARDER
