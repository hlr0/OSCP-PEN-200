# Change Log

## v.0.7.0 - 15-09-2024

* Updated IP and CIDR handling logic
* Updated LDAP query logic
* Added additional groups and logic for when parsing for interesting users from module dumps
* Fixed SessionHunter output to not return the created Class
* Added layered encryption to the new module additions
* Tidied up some others bits around the command and module logic section
* Added Toast notification for password spraying
* Added ticket expiry time to module tgtdeleg
* Added -Module VMCheck. Simple Module that checks if the remote system is a VM or physical system
* Fixed an issue where inaccurate data for machine accounts may be parsed with -Module eKeys

## v.0.6.9 - 07-08-2024

Added `-Elevate` switch which can be combined with `-Command` to run elevated commands against targets.

Added various new modules:

<table><thead><tr><th width="335">Module</th><th>Description</th></tr></thead><tbody><tr><td><a href="https://viperone.gitbook.io/pentest-everything/psmapexec/modules/ntlm">NTLM</a></td><td>Obtains a NTLM hash for each user logon session</td></tr><tr><td><a href="https://viperone.gitbook.io/pentest-everything/psmapexec/modules/sessionexec">SessionExec</a></td><td>Execute a command as each user logon session</td></tr><tr><td><a href="https://viperone.gitbook.io/pentest-everything/psmapexec/modules/sessionrelay">SessionRelay</a></td><td>Capture NTLM hashes for each user logon session</td></tr><tr><td><a href="https://viperone.gitbook.io/pentest-everything/psmapexec/modules/tgtdeleg">TGTDeleg</a></td><td>Grab a TGT for each user logon session</td></tr></tbody></table>

## v.0.6.5 - 13-07-2024

Added various new modules as shown in the table below:

| Module                                                                                  | Description                 |
| --------------------------------------------------------------------------------------- | --------------------------- |
| [FileZilla](https://viperone.gitbook.io/pentest-everything/psmapexec/modules/filezilla) | Dumps FileZilla credentials |
| [Notepad](https://viperone.gitbook.io/pentest-everything/psmapexec/modules/notepad)     | Dumps notepad backup files  |
| [VNC](https://viperone.gitbook.io/pentest-everything/psmapexec/modules/vnc)             | Dumps VNC credentials       |
| [Wi-Fi](https://viperone.gitbook.io/pentest-everything/psmapexec/modules/wi-fi)         | Dumps Wi-Fi credentials     |
| [WinSCP](https://viperone.gitbook.io/pentest-everything/psmapexec/modules/winscp)       | Dumps WinSCP credentials    |

**WinRM / SAM**

Resolved an issue with the `-Module SAM` not retrieving output when using `-Method WinRM`.

**NTDS Module**

Made some adjustments to the NTDS module parsing function. After dumping NTDS with `-Module NTDS` the following files will be created:

* Full NTDS dump in hashcat format
* All computer hashes
* All user hashes
* Hashes for all enabled users
* Users with empty passwords
* Enabled users with empty passwords
* User accounts grouped by password reuse
* Enabled user accounts grouped by password reuse
* User accounts with passwords matching their usernames"

<figure><img src="../.gitbook/assets/image (2165).png" alt=""><figcaption></figcaption></figure>





## v.0.6.2 - 09-06-2024



* Fixed an issue where if an Active Directory users SamAccountName property did not match that provided with -Username, authentication would fail.
* Added checks to ensure when dumping LogonPasswords with `-Module LogonPasswords`, local administrator credentials are no longer reported as Domain Administrator or Enterprise Administrator findings
* Added `-module SCCM`. Uses a stripped down and revised version of SharpSCCM to dump local SCCM Network Access Account credentials and Task sequences. Results are parsed after execution for interesting data.

## v.0.6.0 - 01-05-2024

PsMapExec now no longer requires pulling some modules and scripts from Github. It can now be run fully in restricted and exam / CTF environments.

Version 0.6.0 also supports a custom embedded `Mimikatz` that is much less likely to be detected by AV and which also supports newer versions of Windows (Windows 11, Server 2019 and Server 2022). &#x20;

Otherwise, some smaller changes have been made:

* Fixed a syntax issue in BloodHound query generation that was causing syntax errors on principal names with special characters in their value
* Added initial encryption support for when executing commands on remote systems. Mostly, this is for helping to get round string based AV signatures when running well known scripts.&#x20;
* Added warning output to  `-method WMI` for when classes are unable to be cleaned up
* Removed the `-module tickets` as `-module KerbDump` is far superior in output and much less detectable with AV

## v.0.5.3 - 06-03-2024

Some small fixes and additions

* Added prevention of parsing functions running when combined with an incompatible method
* Added support for environments where RC4 Kerberos types and NTLM hashes are not supported.
  * Reports when attempting to inject a ticket and the Kerberos etype is not supported
  * Will produce a similar output when spraying RC4 hashes against any valid accounts
  * add the Rubeus /enctype:aes256 parameters to improve function in more restricted environments
* Fixed an issue where successful spraying results were being duplicated in output files
* Fixed an issue with IPMI not accepting DNS hostnames
* Added a built in query for "AdminCount=1" when spraying. Useful in large environments where you want to filter down spraying results to those that are likely to be interesting

## v.0.5.2 - 06-03-2024

Small update to address a couple of small issues or quality of life improvements.

* Invalid principal names will be reported when using -Username without -LocalAuth
* The Method IPMI now supports single user targeting
* Local IP addresses from the running system will be excluded from CIDR ranges
* Updated the embedded AMSI bypass so it is no longer detected by Defender

## v.0.5.1 - 27-02-2024

Three new methods added to this update

**Inject:** Used to inject a ticket into memory, useful when working from a non-domain joined system or wish to "revert" to the new injected ticket after performing impersonation

```powershell
# Inject using a Base64 encoded ticket
PsMapExec -Method Inject -Ticket "doIhsj..."
PsMapExec -Method Inject -Ticket "C:\ticket.txt"

# Inject using a hash (RC4/AES256/NTLM)
PsMapExec -Method Inject -Username [User] -Hash [Hash]

# Inject using a password
PsMapExec Method Inject -Username [User] -Password [Password]
```

{% content-ref url="methods/inject.md" %}
[inject.md](methods/inject.md)
{% endcontent-ref %}



**Kerberoast:** Kerberoasting is now possible

Kerberoasting can be performed against the current or a specified domain. By default all users with SPNs will be processed otherwise, it is also possible to select a singular user to kerberoast

```powershell
# Grab all kerberoastable users from current domain
PsMapExec -Method Kerberoast

# Grab all kerberoastable users from a specified domain
PsMapExec -Method Kerberoast -Domain Security.local

# Grab a single specified users from the current domain
PsMapExec -Method Kerberoast -Option Kerberoast:USER
```

{% content-ref url="methods/kerberoast.md" %}
[kerberoast.md](methods/kerberoast.md)
{% endcontent-ref %}



**IPMI:** It is now possible to dump IPMI hashes

<pre class="language-powershell"><code class="lang-powershell"><strong># Check for IPMI in 192.168.56.0/24
</strong><strong>PsMapExec -Targets [192.168.56.0/24] -Method IPMI
</strong>
# Dump IPMI against all Servers in AD, with a user list generated from all domain users
PsMapExec -Targets [Servers] -Method IPMI -Option IPMI:DomainUsers
</code></pre>

{% content-ref url="methods/ipmi.md" %}
[ipmi.md](methods/ipmi.md)
{% endcontent-ref %}

**RDP**

Small update to the RDP method. This will now output a log file by the name of the user which contains all systems for which they have RDP access to. This file is output to $PWD\PME\RDP\\

**Spray**

The Spray method now advises which users were successfully sprayed once it has finished performing the spray

**Target Acquisition**

PsMapExec now supports single IP address and CIDR ranges as target specifications. This is much less preferable (various reasons) than using the built in Active Directory queries (All, Servers, Workstations, DCs).&#x20;

```powershell
# Single IP
PsMapExec -Targets 192.168.56.11 -Method WMI -Command "net user"

# CIDR
PsMapExec -Targets 192.168.56.0/24 -Methid WMI -Command "net user"
```

**Other**

* Resolved an issue where LDAP queries would become "polluted" when querying objects in one domain and then next command querying those in another domain
* Added a small check to ensure -Module or -Commands are not populated when using -Method All. This method is used for checking access across several methods and does not support code execution
* Remove an LDAP binding to the domain PDC after restoring the originating users kerberos ticket. It was causing an issue where sometimes it would wipe over the ticket cache for some reason.
* Made a small change to the parsing functions for -Module SAM and -Module LogonPasswords to accomodate when IP addresses are used for target specification
* Added a small line to ensure the -Module LogonPasswords creates a file for all collected unique hashes

## v.0.4.7 - 24-01-2024

This update includes some changes, reduction and obfuscation to some of the code within the script to help bypass AV. Tested against Windows Defender and Sophos and able to run without using AMSI bypasses. Obviously this gets caught by decent EDR such as Crowdstrike.



**GenRelayList**&#x20;

Changed code to be a revision of: [https://github.com/tmenochet/PowerScan/blob/master/Recon/Get-SmbStatus.ps1](https://github.com/tmenochet/PowerScan/blob/master/Recon/Get-SmbStatus.ps1), Uses a similar code base to the original GenRelayList but is smaller and more effecient.

**RDP**\
Made some changes to the RDP method and added some obfuscation to aid with AV evasion

**Rubeus**

Replaced the Rubeus binary within the script to the most recent version of Rubeus, removed functions from the main code that are not used within PsMapExec and performed some general obfuscation on the binary.&#x20;

**Targets**

Made a change to the target acquisition within the script. Previously, defining `-Targets` Servers would grab all windows server operating systems from the domain, which included Domain Controllers. The Domain Controllers have now been redacted from these results. To now target Domain Controllers use either -Targets All, -Targets DCs or -Targets \[DC Name]

**NTLM hashes**

NTLM hashes can now be provided to -Hash and -SprayHash in the format `NT:LM`.

```powershell
PsMapExec -Targets all -Method wmi -Hash aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe

# Spraying
PsMapExec -Method spray -SprayHash aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe
```

**Other**

* The SAM module has been updated to not show the "Hashes valid on multiple computers" table if no results are actually returned
* Fixed an issue with the hash spraying method that was preventing any results being displayed
* Ammended the regex pattern used in `-Rainbow` to match the new way NTLM.pw returns results.
* Fixed an issue that was preventing the Amnesiac module from working as expected when credentials are supplied

## v.0.4.6 - 13-12-2023

**Amnesiac Module**

Support for the C2 tool Amnesiac has been added. We have colabrated closely with Leo4j to implement Amnesiac into PsMapExec.&#x20;

Amnesiac is an in memory based PowerShell C2 allowing us to spray shells across a domain and gain persistence with Amnesiac to perform further exploitation of targets.

PsMapExec will automatically download and execute Amnesiac in a new process and execute the required payloads on the specified targets systems to gain persistent shells back.

The Amnesiac module works across all command execution methods with PsMapExec.&#x20;

<figure><img src="../.gitbook/assets/image (2113).png" alt=""><figcaption></figcaption></figure>

Some usage examples:

<pre class="language-powershell"><code class="lang-powershell"><strong># Execute over WMI as the current User
</strong><strong>PsMapExec -Targets All -Method WMI -Module Amnesiac
</strong><strong>
</strong><strong># Execute over WinRM using a provided username and hash
</strong><strong>PsMapExec -Targets All -Method WinRM -Module Amnesiac -Username User -Hash [RC4/AES256]
</strong><strong>
</strong><strong># Execute over SMB using a provided username and password
</strong><strong>PsMapExec -Targets All -Method SMB -Module Amnesiac -Username User -Password Pass
</strong></code></pre>

Amnesiac:  [https://github.com/Leo4j/Amnesiac](https://github.com/Leo4j/Amnesiac)\
Documentation: [https://leo4j.gitbook.io/amnesiac/](https://leo4j.gitbook.io/amnesiac/)

**Rainbow Tables**

* PsMapExec now supports rainbow table lookup against an online database (https://ntlm.pw) when parsing extracted hashes from certain modules (SAM, NTDS, LogonPasswords).
* Use the switch `-Rainbow` along side one of these supported modules to perform the check
* Only known hashes will be printed to the console, invalid results are surpressed

```powershell
PsMapExec -Targets Servers -Method WMI -Module SAM -Rainbow
```

<figure><img src="../.gitbook/assets/image (2) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

**LDAP queries**

* Improved logic around obtaining user accounts from high value domain groups. This will now only run when this information is actually required. These are now stored in a global variable so will not be queried in subsequent queries if required after the first time.
* Improved logic around target acquisition. When supplying Servers, Workstations, DCs or All to `-Targets` these results are stored in a global variable. This should improve execution time on larger networks as we will not need to requery the same systems again.&#x20;
* As per both the points above, the `-Flush` switch has been added. This will flush any collected Targets and high value domain group members from memory. Effectively meaning the next time these are run they will be refreshed with infromation from new LDAP queries.

**Password and Hash Spraying**

* Added some error handling so the script does not get stuck in a rare moment querying a user from the domain
* Improved the overall speed of spraying hashes and passwords.

**Other**

* When the original user ticket is restored after performing impersonation a LDAP kerberos ticket will now be created for the Primary Domain Controller in addition to the krbtgt ticket.
* The method GenRelayList has now been moved inside a runspace to support threading
* Added logic to ensure duplicate computer names from a target file are not processed more than once
* Moved the IP resolution for the -Method MSSQL to inside a runspace to speed things up
* Changed the default timeout value on port scans to 3 seconds. This new default value should provide a better balance between speed and result accuracy.
* Removed the parameter ordering so it does not matter which order parameters are now provided to PsMapExec
* added some logic around the "Unable to retrieve kerberos ticket" issue. This is usually caused by no kerberos tickets being in the existing cache. This check will now be skipped if no tickets exist in the current kerberos cache



## v.0.4.5 - 29-11-2023

**Target Acquisition**&#x20;

Targets can now be read from a file. The file should either be a line seperated or a comma seperated list of NETBIOS and / or FQDNs.

```powershell
PsMapExec -Targets "Targets.txt" -Method WMI -Command "whoami"
```

<pre class="language-powershell"><code class="lang-powershell"><strong># File example
</strong><strong>DC01
</strong>CA01.security.local
SQL02.security.local
SRV02
</code></pre>

Targets also now supports wildcard matching

```powershell
PsMapExec -Targets "SRV*" -Method WMI -Command whoami
```

**MSSQL**&#x20;

* Resolved an IP resolution issue in the `-Method MSSQL`
* Resolved an issue where providing a single target to the `-Method MSSQL` that would inadvertently check  all systems with MSSQL SPNs as well
* Made some general improvements and fixes to the method when producing output and checking results.
* Discovered MSSQL instances are now output to `$pwd\PME\MSSQL\`

Accessible and SYSADMIN accessbile MSSQL instances are now output to file:

&#x20;`$pwd\PME\MSSQL\$Username-Accessible-MSSQL-Instances.txt`

`$pwd\PME\MSSQL\$Username-SYSADMIN-Accessible-MSSQL-Instances`



**Other**

* Added switch `-NoBanner`. Prevents the banner from being displayed.
* Added initial support for `-Method All`
* Added the `-Timeout` parameter. This controls the timeout on port scanning across all methods before determining a port is not up on a host. The default value is 100ms.&#x20;
* Improved domain binding function. Should help in those cases where a createnetonly session is created and some issues with reading from the PDC are encountered.
* The parsing function for `-Method LogonPasswords` should also now correctly seperate discovered usernames from output in a cleaner fashion
* Changed the "pre-built" command that is suggested when a TGT is discovered on the kerbdump module to use the `-Targets` and `-Method` value originally used when performing the dump rather than the generic values "All" and "WMI".&#x20;
* Redcued some of the code for displaying results
* Refactored the WMI code to be cleaner and be self contained within a single function as opposed to a complete different function when `-LocalAuth` is provided
* Moved some of the code around so that the "Create directory" text always happens after the "targets designation"  text.
* If no machines are found in the LDAP querying, to terminate the script early
* Removed "MacOS" and "MacOS X" from candidate systems
* Changed the thread count check to produce an error when a `-Thread` value of less than 1 or more than 100 is provided. (Providing a very high thread count can invalidate results). Default value for `-Threads` has been changed to 30.

## v.0.4.3 - 11-11-2023

* Added a new -Module NTDS.&#x20;
* NTDS will perform DCSync against target systems (That are DCs) and dump all NTLM and SAM hashes from the database.&#x20;

Parsing functions are includes to perform the following after extraction:

* Create a file that stores all computer NTLM hashes
* Create a file that stores all user NTLM hashes
* Create a file that stores SAM hashes from the DC
* Create a file that identifies accounts with empty passwords
* Create a file that identifies groups of users with identical passwords

## v.0.4.2 - 09-11-2023

* Added Initial support for the `-Method` MSSQL
* Documentation MSSQL here: [https://viperone.gitbook.io/pentest-everything/psmapexec/mssql](https://viperone.gitbook.io/pentest-everything/psmapexec/mssql)

## v.0.4.1 - 30-10-2023

* Added further support for PsMapExec running from a non-domain joined system against a specified domain

Change some of the syntac for the Spray method to support these changes:

* Added `-SprayHash` instead of `-Hash` when specifying credential material to spray
* Added -SprayPassword instead of `-SprayPassword` when specifying credential material to spray

This allows for the `-Password` and `-Hash` parameters to be used for when authenticating against a domain when working from a non-domain joined system.

## v.0.4.0 - 27-10-2023

* RDP module updated. Updated and improved SharpRDP binary submitted by [https://github.com/hosakauk](https://github.com/hosakauk). Binary now only performs and validates connection requests. Code should be about 10x faster than previous versions
* Updated the way PsMapExec handles threading for RDP.&#x20;
* Cleaned up various small bits and added some error checking.

## v.0.3.8 - 10-10-2023

* Added the method VNC. Can be used to check hosts that have "No Auth" set.&#x20;
* Updated code for parsing eKeys and LogonPasswords. This should now be much cleaner and parse useful information with a higher degree of accuracy

## v.0.3.5 - 4-10-2023

* Small update to address an issue with tickets clearing for the logon session when using Spray or GenRelayList, and the method terminates early.

## v.0.3.4 - 4-10-2023



* SMB Protocol: Optimized some parts of the method, this should now return results much faster.
* eKeys: Parsing for eKeys has been updated. This will now display each username as DOMAIN\USERNAME format. Tags have also been updated to notify when an account with AdminCount=1 has been identified and if an account with an Empty Password has been found.
* Consistent logic for each method has been written when retreiving results from a remote system.
* Updated the files module to correctly work over WinRM and SMB.
* Refactored the SessionHunter method to now only display computers for which we have administrative access to, and for where there is an administrative user session active on the system (Administrative users credentials are likely in memory)

## v.0.3.2 - 25-09-2023

* New Method: PsExec has been removed and replaced with Leo4j's Invoke-SMBRemoting. Use `-Method SMB` to initialise.
* GenRelayList: Resolved an issue with this method which was reporting incorrect SMB signing status on some hosts.
* Code Cleanup: More parts of the code have been generally cleaned up.

<figure><img src="../.gitbook/assets/image (2101).png" alt=""><figcaption></figcaption></figure>

## v.0.3.0 - 22-09-2023

* More speed: We are now using runspaces for WinRM and WMI. Things should be a lot quicker for these two methods now. We also have some additional filtering when port scanning. This should should strip out non-candidate systems even better.&#x20;
* Syntax change: To use SessionHunter,GenRelayList and Spray we now see this as a parameter for `-Method` for example: `PsMapExec -Targets All -Method GenRelayList` . This is to ensure more consistency across syntax.
* Code Cleanup: Various bits of code throughout has been cleaned up and generally just made more efficient. There is a lot more work in this regard to complete however.
