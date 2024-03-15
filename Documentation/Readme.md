# Active Directory Hacking Readme..
---------------------------------------------------------------
---------------------------------------------------------------
---------------------------------------------------------------
# What is Active Directory
### Active Directory Kerberos Authentication Method
---------------------------------------------------------------
### Basic Description:
Active Directory (a.k.a Domain Controller) is a Database with Services that allow a user to connect and use network resources or resources that allow the user to get there job done.

The Active Directory Database is critical aspect as it contains the username and password and hashes and various other aspects like the name, surname, email and etc

The services are the authentication aspect whereby it allows the user to use the service. 

The Username and password is the authorization aspect that allows a user to connect.\
The password is not salted in active directory

### Sites - Replication
Think of two buildings with a road connecting those two buildings.\
In both buildings you have 5 domain controllers each .. (you can have as many as you want for the setup requirements)\
in the first building between those 5 Domain Controllers you have something called replication between those 5 domain controllers\
If for example 2 go down - the other 3 will pick up the slack and continue working\
This is known as INTRASITE REPLICATION

The road between the two buildings is known as INTERSITE REPLICATION so that it can replicate\
between both buildings - making Active Directory a diverse setup and stable.


### Forests, Trees, Trust
Lets take one Domain Controller - this is known as a forest and your first Domain Controller\
is known as a Global Catalogue which setups up the forest.\
Microsoft states that the forest is a security boundary.\
Objects in different Active Directory forests are not able to interact with each other unless\ 
the administrators of each forest create a trust between them.\
Each Forest (yes you can have more than one forest in a organization setup)\
will contain domains also known as trees. 

Each Tree Aspect has something known as a domain. You can get a Parent domain\
and a child domain as many as you want. A domain is basicaly the organization name for example\
google.com and its child will be samurai.google.com.

### Trust
In Trust the two most common are one way or two way trust. Whereby a child can have one way trust to\
its parent or parent to child two way trust. You can also have trust between a child of parent for one\
domain connecting to the trust be it one way or two way for another domain say - ospc.com

### Domain, Organization Unit, Group policies
In each domain you have something called organization units. You can have as many as you want.\
Basically a organization unit is for example the various departments in a company.\
Example being - IT_staff / Staff / Marketing / Management / HR / Sales and so forth\
Now you cannot add a group policy to a domain (not good practice). but you can add a group policy\
to the organizational unit. A group policy is basically where you do not want say upper management\
to change there background image on there computers.\

### Objects, Attributes, Attributes Values
In each organizational unit you have something called objects which contain for example\
users and computers. which users are assocatied to. Each object has attributes assocaited to it\
which also have attribute values also associated to it.

### Permissions
In Active Directory you have something called permissions. Basically permissions is whereby\
you have a user that wants access to a folder within a organization unit say HR that wants\
read, write and execute permissions on the folder to do there work.


### SYSVOL folder
One of these methods is mining SYSVOL for credential data - login and group policy.\
SYSVOL contains logon scripts, group policy data, and other domain-wide data which needs to be available\
SYSVOL is the domain-wide share in Active Directory to which all authenticated users have read access.\
anywhere there is a Domain Controller (since SYSVOL is automatically synchronized and shared among all Domain Controllers).\
but placing a file in it will not get executed. That would be the group policy to allow execution of the file.\

All domain Group Policies are stored here: \\<DOMAIN>\SYSVOL\<DOMAIN>\Policies\

---------------------------------------------------------------
---------------------------------------------------------------
---------------------------------------------------------------
---------------------------------------------------------------
# 10 CONCEPTS FOR ACTIVE DIRECTORY 
Master these 10 Terms/Concepts and you'll be a Beast!
---------------------------------------------------------------

### 1. DCSYNC/DSReplication_GetChangesAll/AllowedToDelegate/Replication
##### Description: 
This is number one because its the most fun. DCSync allows an attacker to impersonate a Failover Domain Controller. In that context, the production Domain Controller shares all user hashes upon request, ergo DCSYNC. GetChangesAll, Replication and AllowedToDelegate all point toward the possibility of DCSYNC.

Environment: Forest/Sizzle 

### 2. ByPassing AMSI - "Antimalware Scan Interface"
##### Description: 
It may be necessary to bypass the anti-virus in Active Directory. Attackers can attempt to bypass AMSI with the Bypass-4MSI command in Evil-WinRM. Always run this command before introducing a malicious script to the environment. 

Environment: Forest from Hackthebox

### 3. WriteDACL/Account Operators
##### Description:
In the account operators group, an attacker can create users and place them in non-protected groups. Placing a new user in a group with WriteDACL, enables an attacker to modify the new user's DACL. In this example, we give our new user DCSync rights.\
Discretionary Access Control Lists (DACLs) and Acccess Control Entries (ACEs) that make up DACLs.
Active Directory objects such as users and groups are securable objects and DACL/ACEs define who can read/modify those objects (i.e change account name, reset password, etc). 

Environment: Forest from Hackthebox 

### 4. NTDS.dit and System.hive
##### Description: 
With these files and the appropriate permissions, an attacker can dump hashes from the Domain Controller using DCSync.

Environment: Blackfield from Hackthebox

### 5. SeBackupPrivilege and SeRestorePrivilege
##### Description: 
SeBackupPrivilege and SeRestorePrivilege allows the attacker access to any file on the machine given he/her takes the appropriate steps. In this example, we acquire NTDS.dit and System.hive

Environment: Blackfield from Hackthebox

### 6. WriteOwner
##### Description: 
WriteOwner permissions allows an attacker to set the owner of the object and make him/herself a member of the object.

Environment: Object from HackTheBox

### 7. PowerView / net.exe
##### Description: 
Allows for additional manipulation of Active Directory. Many of the commands presented by BloodHound require PowerView.
The net.exe is a built-in tool in Windows that can be used to carry out tasks on groups, users, accounts, policies, etc. [17]. Using the net.exe tool, the attacker can see all the attributes of users and groups and hunt for critical groups such as AD domain admins and can then see information about the user’s group membership.

Environment: Object from Hackthebox

### 8. ForceChangePassword 
##### Description: 
ForceChangePassword allows an attacker to change the password of the object in question.

Environment: Object from Hackthebox

### 9. GenericWrite/GenericAll/AllExtendedRights 
##### Description: 
GenericAll allows an attacker to modify the object in question. In this example, we change the password of a Domain Administrator. GenericWrite allows the modification of certain things (More on this in Object from Hackthebox).

Environment: Search from HacktheBox

### 10. ReadGMSAPassword 
##### Description: 
ReadGMSAPassword allows an attacker to use the password of a Group Managed Service Account which usually has elevated privileges. 

Environment: Search from HacktheBox

---------------------------------------------------------------
---------------------------------------------------------------
---------------------------------------------------------------
# Methodology
---------------------------------------------------------------
- Domain Enumeration = get all information
- Escalate Priveleges = hunting for local or admin privileges
- Persistance = make kerberoast or asreproasting
- Data exfiltration = denial of service, or any other target

# Enumeration Methodology

- ### Step 1 - Information Gathering
  - Ip Addresses and domain users
  - Subnet information
  - Network topology
  - Domain Controller details
  - Dns server information
       
- ### Step 2 - Enumerate Users
  - Nmap Scanning: Use Nmap to identify open ports on the target systems, particularly the domain controllers. Run a command like nmap -p 139,445 -T4 -v -oA nmap_scan <target>.
  - NetBIOS Enumeration: Use tools like enum4linux or nbtscan to enumerate NetBIOS information, including users and shares.
  - LDAP Enumeration: Enumerate users and groups using LDAP queries. Tools like ldapsearch can be handy. For example: ldapsearch -x -h <domain_controller> -b "dc=<target_dc>,dc=com" -D "<your_user>" -W.
 
- ### Step 3 - Enumerate Groups
  - Net Group Enumeration: Utilize the net command to enumerate groups. For example: net group /domain.
  - PowerShell Enumeration: Run PowerShell scripts to list all Active Directory groups. For example: Get-ADGroup -Filter * | Select-Object Name.

- ### Step 4 - Enumerate Shares & Permissions
  - Enum4linux: Use enum4linux to enumerate shares, SIDs, and permissions on the target system.
  - Accesschk: Run tools like accesschk to check for misconfigured permissions and find vulnerabilities.
    
- ### Step 5 - Enumerate Resources
  - SMB Shares Enumeration: Use tools like smbclient or smbmap to list accessible SMB shares.
  - Kerberos Enumeration: Enumerate service principals using tools like Kerbrute to identify potential attack vectors.

- ### Step 6 - Enumerate Trust Relationships
  - Use the nltest or netdom command to identify trust relationships between domains.

- ### Step 7 - Document and Report
  - always remember to take screenshots and document findings

---------------------------------------------------------------
---------------------------------------------------------------
---------------------------------------------------------------
---------------------------------------------------------------
# Tools for Active Directory

Learn to use the below tools

- smbmap
- smbclient
- Crackmapexec / psexec
- rpcclient
- Ldapsearch
- enum4linux
- kerbrute
- impacket-getnpusers
- impacket-getusersspn
- Evil-winrm
- xfreerdp
- *Mimikatz*
- Rubeus
- Bloodhound


---------------------------------------------------------------
---------------------------------------------------------------
---------------------------------------------------------------
---------------------------------------------------------------
# ATTACK VECTORS ACTIVE DIRECTORY

- ### KERBEROASTING
  - When you think of Kerberoasting think - user has a SPN - ""Service Prinicipal Name"" associated with it - then crack the hash - need access to domain or domain creditionals 
- ### ASREPROASTING 
  - When you think of ASREPROASTING think - Kerberos Pre-authentication is disabled - then crack the hash
- ### LLMNR
  - ***
- ### PASSWORD SPRAYING
  - ***
- ### LSASS
  - ***

