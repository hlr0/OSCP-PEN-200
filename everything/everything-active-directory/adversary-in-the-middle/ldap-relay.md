# 🔨 LDAP Relay

## Description

LDAP Relay attacks make use of NTLM authentication where an NTLM authentication request is performed and an attacker captures the credentials and relays them to a Domain Controller and leverages this against LDAP.

By default without LDAP signing and channel binding this attack is possible.

## Scenario #1 \[ntlmrelayx]

* Domain Controller - 10.10.10.10 - Windows Server 2019
* Admin-PC 10.10.10.7 - Windows 10 21H1
* Attacker-PC 10.10.10.6 - Kali Linux
* We already have credentials for the user Dave

This scenario also assumes we have valid credentials for a Domain user.

### Attacker-PC

Firstly, ntlmrelayx.py it started on the attacking machine with the --escalate-user switch.

```python
sudo python2 ntlmrelayx.py -t ldap://10.10.10.10 --escalate-user Dave
```

![](<../../../.gitbook/assets/image (1936).png>)

### Admin-PC

Through means of phishing or placing a malicious file locally from some other method we are able to trick the Domain Administrator user who is currently logged into Admin-PC into browsing to http://10.10.10.6/.

![](<../../../.gitbook/assets/image (1937).png>)

### Attacker-PC

Where credentials are requested and the Domain Administrator successfully submits them. `ntlmrelayx.py` receives the credentials and relays them to the Domain Controller on 10.10.10.10.

![](<../../../.gitbook/assets/image (1938).png>)

Successfully giving the user Dave DCSync rights which can be used to dump hashes for the Domain with `secretsdump.py`.

```bash
sudo python2 secretsdump.py security/Dave:'Password123!'@10.10.10.10
```

![](<../../../.gitbook/assets/image (1939).png>)

The hashes can then either be taken for password cracking or can be used for authentication. By default Domain Contollers run the `WinRM` service on port 5985. This can be used alongside `Evil-WinRM` to authenticate with hashes.

Below is a example of the dumped hashed for the Domain Administrator being used for access over `WinRM` to the Domain Controller DC01.

```bash
evil-winrm -i 10.10.10.10  -u 'Administrator' -H '0c564715b06bb035a87e91b9f71488a3'
```

![](<../../../.gitbook/assets/image (1940) (1).png>)

## Scenario #2 \[Intercept-ng]

* Domain Controller - 10.10.10.10
* Admin-PC - 10.10.10.7
* Attacker-PC 10.10.10.5

### Attacker-PC

This first example shows the LDAP Relay attack using Intercept-ng on a Windows 10 host. Intercept-ng is started on the system Attacker-PC and an administrator PC 'Admin-PC' is quickly identified.

![](../../../.gitbook/assets/Intercept-ng\_scan.PNG)

The system is then added to the target list on `Intercept-ng`. From here Expert mode is then used on `Intercept-ng` to enable LDAP relay and enter the Domain name as shown below.

![](<../../../.gitbook/assets/image (1918).png>)

Then under the MiTM options LDAP relay is checked and the Domain Controller and Gateway fields are populated with the expected information.

![](<../../../.gitbook/assets/image (1919).png>)

From here the MiTM attack is started and active sniffing begins. As shown below we are given the Inject URL which is `http://Attacker-PC:62222/`.

![](<../../../.gitbook/assets/image (1920).png>)

### Admin-PC

Through means of phishing or placing a malicious file locally from some other method we are able to trick the Domain Administrator user who is currently logged into Admin-PC into browsing to [http://Attacker-PC:62222/](http://attacker-pc:62222).

![](<../../../.gitbook/assets/image (1921).png>)

### Attacker-PC

Once this Domain Administrator on Admin-PC clicks the malicious link we see Intercept-ng relays the credentials to the Domain Controller and creates a new user called 'cepter'.

![](<../../../.gitbook/assets/image (1922).png>)

### Domain Controller - DC01

Over on our Windows Server 2019 Domain Controller we now see the user 'cepter' exists and is a member of the Domain Admins Group.

![](<../../../.gitbook/assets/image (1923).png>)

### Limitations

Unfortunately though this method Intercept-ng is unable to create the new user account with a password. One way which this can be leveraged however is through connecting to an RDP session to provide the opportunity for a password reset.

![](<../../../.gitbook/assets/image (1944).png>)

Changing the password from here will provide the new Domain Administrator account with a valid password we can use across the network. The limitation in this regard is that Remote Desktop Network Level Authentication needs to be disabled on the RDP host as shown below.

![](<../../../.gitbook/assets/image (1945).png>)

## Mitigations

### LDAP Channel Binding

**\[WIP]**

### LDAP Signing

LDAP signing is the digital signing of LDAP traffic by the source. The digital signing of LDAP traffic guarantees the authenticity and integrity of the contents of the LDAP traffic has not been altered in transit and allows the receiving party to verify the origin of the LDAP traffic. The LDAP signing configuration can be done by using specific group policies or by using registry keys. It is important to note that LDAP signing must be configured on both the domain controllers and clients:

**How to set the server LDAP signing requirement Group Policy Object**

1. Select **Default Domain Controller Policy** > **Computer Configuration** > **Policies** > **Windows Settings** > **Security Settings** > **Local Policies**, and then select **Security Options**.
2. Right-click **Domain controller: LDAP server signing requirements**, and then select **Properties**.
3. In the **Domain controller: LDAP server signing requirements Properties** dialog box, enable **Define this policy setting**, select **Require signing** in the **Define this policy setting** list, and then select **OK**.
4. In the **Confirm Setting Change** dialog box, select **Yes**.

\*\*How to set the client LDAP signing requirement by using local computer policy \*\*

1. Select **Local Computer Policy** > **Computer Configuration** > **Policies** > **Windows Settings** > **Security Settings** > **Local Policies**, and then select **Security Options**.
2. Right-click **Network security: LDAP client signing requirements**, and then select **Properties**.
3. In the **Network security: LDAP client signing requirements Properties** dialog box, select **Require signing** in the list, and then select **OK**.
4. In the **Confirm Setting Change** dialog box, select **Yes**.

**How to set the client LDAP signing requirement by using a domain Group Policy Object**

1. Select **Default Domain Policy** > **Computer Configuration** > **Windows Settings** > **Security Settings** > **Local Policies**, and then select **Security Options**.
2. In the **Network security: LDAP client signing requirements Properties** dialog box, select **Require signing** in the list, and then select **OK**.
3. In the **Confirm Setting Change** dialog box, select **Yes**.

![](<../../../.gitbook/assets/image (1941).png>)

### Verifying changes: Confirming with a simple Bind

**How to verify configuration changes**

1. Sign in to a computer that has the AD DS Admin Tools installed.
2. Select **Start** > **Run**, type _ldp.exe_, and then select **OK**.
3. Select **Connection** > **Connect**.
4.  In **Server** and in **Port**, type the server name and the non-SSL/TLS port of your directory server, and then select **OK**.

    Note

    For an Active Directory Domain Controller, the applicable port is 389.
5. After a connection is established, select **Connection** > **Bind**.
6. Under **Bind type**, select **Simple bind**.
7.  Type the user name and password, and then select **OK**.

    If you receive the following error message, you have successfully configured your directory server:

    > Ldap\_simple\_bind\_s() failed: Strong Authentication Required

![](<../../../.gitbook/assets/image (1943).png>)

### Verifying changes: Attack perspective

Attempting the same attack again after the required Group Policy changes for `LDAP` signing have been made produces the following result on the attackers system.

![](<../../../.gitbook/assets/image (1942).png>)

## Resources:

{% embed url="https://blog.fox-it.com/2018/04/26/escalating-privileges-with-acls-in-active-directory/" %}

{% embed url="https://oxfordcomputergroup.com/resources/ldap-channel-binding-signing-requirements/" %}

{% embed url="https://docs.microsoft.com/en-us/troubleshoot/windows-server/identity/enable-ldap-signing-in-windows-server" %}

{% embed url="https://support.microsoft.com/en-us/topic/use-the-ldapenforcechannelbinding-registry-entry-to-make-ldap-authentication-over-ssl-tls-more-secure-e9ecfa27-5e57-8519-6ba3-d2c06b21812e" %}
