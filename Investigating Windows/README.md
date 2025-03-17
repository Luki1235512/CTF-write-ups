# Investigating Windows

## A windows machine has been hacked, its your job to go investigate this windows machine and find clues to what the hacker might have done.

### Whats the version and year of the windows machine

```Powershell
Get-ComputerInfo | Select-Object WindowsProductName
```

[SCREEN01]

### Wich user logged in last?

```Powershell
Get-LocalUser | Where-Object {$_.LastLogon -ne $null} | Sort-Object -Property LastLogon -Descending | Select-Object Name, LastLogon -First 1
```

[SCREEN02]

### When did John log onto the system last?

```Powershell
Get-LocalUser -Name "John" | Select-Object Name, LastLogon
```

[SCREEN03]

### What IP does the system connect to when it first starts?

1. Using PowerShell:

```Powershell
Get-CimInstance Win32_StartupCommand | Select-Object Name, Command, Location, User
```

[SCREEN05]

2. Or open `regedit`, and go to the: HKEY_LOCAL_MACHINE > SOFTWARE > Microsoft > Windows > CurrentVersion > Run

[SCREEN04]

### What two accounts had administrative privileges (other than the Administrator user)?

```Powershell
Get-LocalGroupMember -Group "Administrators" | Select-Object Name, ObjectClass
```

[SCREEN06]

### Whats the name of the scheduled task that is malicous?

```Powershell
Get-ScheduledTask
```

[SCREEN07]

### What file was the task trying to run daily?

### What port did this file listen locally for?

```Powershell
Get-ScheduledTask -TaskName "Clean file system" | Select-Object -ExpandProperty Actions
```

[SCREEN08]

### When did Jenny last logon?

```Powershell
net user Jenny | findstr "Last logon"
```

[SCREEN09]

### At what date did the compromise take place?

All the `Date modified` dates on `inetpub`, `TMP`, `Users` and `Windows` pointing to the same date

[SCREEN10]

### During the compromise, at what time did Windows first assign special privileges to a new logon?

Go to `Event Viewer` -> `Custom Views` -> `Create Custom View`, then look through logs
[SCREEN11]
[SCREEN12]

### What tool was used to get Windows passwords?

Like all the malicious files the answer is in `TMP` file

[SCREEN13]

### What was the attackers external control and command servers IP?

The `google.com` ip looks suspicious

```Powershell
Get-Content c:\Windows\System32\drivers\etc\hosts
```

[SCREEEN14]

### What was the extension name of the shell uploaded via the servers website?

The file is in `C:\inetpub\wwwroot`

[SCREEN15]

### What was the last port the attacker opened?

Go to the `Windows Firewall with Advanced Security` and check rule at the top

[SCREEN16]

### Check for DNS poisoning, what site was targeted?

[SCREEN14]
