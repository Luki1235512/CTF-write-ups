# Investigating Windows

## A windows machine has been hacked, its your job to go investigate this windows machine and find clues to what the hacker might have done.

### Whats the version and year of the windows machine

```Powershell
Get-ComputerInfo | Select-Object WindowsProductName
```

![SCREEN01](https://github.com/user-attachments/assets/e963ce11-cdd2-4a39-aec7-d9a8c334e03d)

### Wich user logged in last?

```Powershell
Get-LocalUser | Where-Object {$_.LastLogon -ne $null} | Sort-Object -Property LastLogon -Descending | Select-Object Name, LastLogon -First 1
```

![SCREEN02](https://github.com/user-attachments/assets/192bcf4e-90c3-45e5-ad6f-c86164e4e1c3)

### When did John log onto the system last?

```Powershell
Get-LocalUser -Name "John" | Select-Object Name, LastLogon
```

![SCREEN03](https://github.com/user-attachments/assets/51bc1a41-1019-488a-8a18-f7342f96542f)

### What IP does the system connect to when it first starts?

1. Using PowerShell:

```Powershell
Get-CimInstance Win32_StartupCommand | Select-Object Name, Command, Location, User
```

![SCREEN05](https://github.com/user-attachments/assets/9574a7db-6303-4291-9e71-cf398bee3c1f)

2. Or open `regedit`, and go to the: HKEY_LOCAL_MACHINE > SOFTWARE > Microsoft > Windows > CurrentVersion > Run

![SCREEN04](https://github.com/user-attachments/assets/2347d0e0-566e-4de5-8681-6c402727fd7a)

### What two accounts had administrative privileges (other than the Administrator user)?

```Powershell
Get-LocalGroupMember -Group "Administrators" | Select-Object Name, ObjectClass
```

![SCREEN06](https://github.com/user-attachments/assets/2967ff2e-8f78-4abd-9afe-e95e88818548)

### Whats the name of the scheduled task that is malicous?

```Powershell
Get-ScheduledTask
```

![SCREEN07](https://github.com/user-attachments/assets/2facb902-37af-4959-b7db-e5e8d1a0c146)

### What file was the task trying to run daily?

### What port did this file listen locally for?

```Powershell
Get-ScheduledTask -TaskName "Clean file system" | Select-Object -ExpandProperty Actions
```

![SCREEN08](https://github.com/user-attachments/assets/c78d8ba4-f7d8-4462-9bde-dc14d0feb6e9)

### When did Jenny last logon?

```Powershell
net user Jenny | findstr "Last logon"
```

![SCREEN09](https://github.com/user-attachments/assets/7c42045a-35af-40a4-84bd-f2d4fe0d7354)

### At what date did the compromise take place?

All the `Date modified` dates on `inetpub`, `TMP`, `Users` and `Windows` pointing to the same date

![SCREEN10](https://github.com/user-attachments/assets/8e7d174b-206f-4471-96ed-dff54e287abb)

### During the compromise, at what time did Windows first assign special privileges to a new logon?

Go to `Event Viewer` -> `Custom Views` -> `Create Custom View`, then look through logs

![SCREEN11](https://github.com/user-attachments/assets/035a8b34-d46f-41f2-8741-fd300e5b036a)
![SCREEN12](https://github.com/user-attachments/assets/d813cebc-04d8-476e-ada5-a3a5f827faa9)

### What tool was used to get Windows passwords?

Like all the malicious files the answer is in `TMP` file

![SCREEN13](https://github.com/user-attachments/assets/480fb0cc-81fb-4ac0-87b2-33a5a2ea1ff9)

### What was the attackers external control and command servers IP?

The `google.com` ip looks suspicious

```Powershell
Get-Content c:\Windows\System32\drivers\etc\hosts
```

![SCREEN14](https://github.com/user-attachments/assets/aecf09b0-ff34-453b-a794-8f05e7f35a8b)

### What was the extension name of the shell uploaded via the servers website?

The file is in `C:\inetpub\wwwroot`

![SCREEN15](https://github.com/user-attachments/assets/16c2ab79-45ee-4dbe-9660-db30f09de255)

### What was the last port the attacker opened?

Go to the `Windows Firewall with Advanced Security` and check rule at the top

![SCREEN16](https://github.com/user-attachments/assets/e7da3326-24fd-4f82-b041-3da555342080)

### Check for DNS poisoning, what site was targeted?

![SCREEN14](https://github.com/user-attachments/assets/9b7d0847-57c1-4d64-8181-a251e0a6a810)
