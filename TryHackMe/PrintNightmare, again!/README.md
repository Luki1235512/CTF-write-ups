# [PrintNightmare, again!](https://tryhackme.com/room/printnightmarec2bn7l)

## Search the artifacts on the endpoint to determine if the employee used any of the Windows Printer Spooler vulnerabilities to elevate their privileges.

**Scenario:** In the weekly internal security meeting it was reported that an employee overheard two co-workers discussing the PrintNightmare exploit and how they can use it to elevate their privileges on their local computers.

**Task:** Inspect the artifacts on the endpoint to detect the exploit they used.

Note: Use the **FullEventLogView** tool. Go to `Options > Advanced Options` and set `Show events from all times`.

### The user downloaded a zip file. What was the zip file saved as?

_You can try using ProcDOT or FullEventLogView_

1. Go to `Edit > Find` and search for `.zip`

[SCREEN01]

**Answer:** `levelup.zip`

---

### What is the full path to the exploit the user executed?

1. Go to `Edit > Find` and search for `Execute Remote Command`

[SCREEN02]

**Answer:** `C:\Users\bmurphy\Downloads\CVE-2021-1675-main\CVE-2021-1675.ps1`

---

### What is the full path to the exploit the user executed?

1. Go to `Edit > Find` and search for `RuleName: DLL`

[SCREEN03]

**Answer:** `C:\Users\bmurphy\AppData\Local\Temp\3\nightmare.dll`

---

### What was the full location the DLL loads from?

1. Go to `Edit > Find` and search for `nightmare.dll`

[SCREEN04]

**Answer:** `C:\Windows\system32\spool\DRIVERS\x64\3\nightmare.dll`

---

### What is the primary registry path associated with this attack?

1. Go to `Edit > Find` and search for `CVE-2021-1675.ps1`

2. Find event from `8/27/2021 2:53:38 AM.956`, then go 3 events down to event ID `13`

[SCREEN05]

**Answer:** `HKLM\System\CurrentControlSet\Control\Print\Environments\Windows x64\Drivers\Version-3\THMPrinter\`

---

### What was the PID for the process that would have been blocked from loading a non-Microsoft-signed binary?

_Microsoft-Windows-Security-Mitigations_

1. Go to `Edit > Find` and search for `would have been blocked from loading`

[SCREEN06]

**Answer:** `2600`

---

### What is the username of the newly created local administrator account?

1. Go to `Edit > Find` and search for `A user account was created`

2. Find event from `8/27/2021 2:53:38 AM.626`

[SCREEN07]

**Answer:** `backup`

---

### What is the password for this user?

_Use ProcDOT to search for PowerShell History File_

1. Open `C:\Users\bmurphy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`

[SCREEN08]

**Answer:** `ucGGDMyFHkqMRWwHtQ`

---

### What two commands did the user execute to cover their tracks? (no space after the comma)

1. Same file as above

**Answer** `rmdir .\CVE-2021-1675-main\,del .\levelup.zip`
