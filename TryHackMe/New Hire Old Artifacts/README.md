# [New Hire Old Artifacts](https://tryhackme.com/room/newhireoldartifacts)

## Investigate the intrusion attack using Splunk.

**Scenario:** You are a SOC Analyst for an MSSP (managed Security Service Provider) company called TryNotHackMe.

A newly acquired customer (Widget LLC) was recently onboarded with the managed Splunk service. The sensor is live, and all the endpoint events are now visible on TryNotHackMe's end. Widget LLC has some concerns with the endpoints in the Finance Dept, especially an endpoint for a recently hired Financial Analyst. The concern is that there was a period (December 2021) when the endpoint security product was turned off, but an official investigation was never conducted.

Your manager has tasked you to sift through the events of Widget LLC's Splunk instance to see if there is anything that the customer needs to be alerted on.

Happy Hunting!

### A Web Browser Password Viewer executed on the infected machine. What is the name of the binary? Enter the full path.

1. Search for `index=* "password viewer"`

[SCREEN01]

**Answer:** `C:\Users\FINANC~1\AppData\Local\Temp\11111.exe`

---

### What is listed as the company name?

1. The same search `index=* "password viewer"`

[SCREEN02]

**Answer:** `NirSoft`

---

### Another suspicious binary running from the same folder was executed on the workstation. What was the name of the binary? What is listed as its original filename? (format: file.xyz,file.xyz)

_File path should include username in long name format. https://docs.microsoft.com/en-us/windows/win32/fileio/naming-a-file_

1. Search for `index=* "C:\\Users\\Finance01\\AppData\\Local\\Temp\\*.exe" OriginalFileName="*"`

[SCREEN03]

**Answer:** `IonicLarge.exe,PalitExplorer.exe`

---

### The binary from the previous question made two outbound connections to a malicious IP address. What was the IP address? Enter the answer in a defang format.

_Cyberchef can help with defanging._

1. Search for `index=* IonicLarge.exe` and check `DestinationIp` field

[SCREEN04]

**Answer:** `2[.]56[.]59[.]42`

---

### The same binary made some change to a registry key. What was the key path?

1. Search for `index=* IonicLarge.exe Registry`

[SCREEN05]

**Answer:** `HKLM\SOFTWARE\Policies\Microsoft\Windows Defender`

---

### Some processes were killed and the associated binaries were deleted. What were the names of the two binaries? (format: file.xyz,file.xyz)

_Process were killed with 'taskkill /im'_

1. Search for `index=* "taskkill /im"`

[SCREEN06]

**Answer:** `WvmIOrcfsuILdX6SNwIRmGOJ.exe,phcIAmLJMAIMSa9j9MpgJo1m.exe`

---

### The attacker ran several commands within a PowerShell session to change the behaviour of Windows Defender. What was the last command executed in the series of similar commands?

_The last command issue has the most recent time stamp_

1. Search for `index=* powershell commandline`

[SCREEN07]

**Answer:** `powershell WMIC /NAMESPACE:\\root\Microsoft\Windows\Defender PATH MSFT_MpPreference call Add ThreatIDDefaultAction_Ids=2147737394 ThreatIDDefaultAction_Actions=6 Force=True`

---

### Based on the previous answer, what were the four IDs set by the attacker? Enter the answer in order of execution. (format: 1st,2nd,3rd,4th)

1. Search for `index=* "PATH MSFT_MpPreference call Add ThreatIDDefaultAction_Ids=*"`

**Answer:** `2147735503,2147737010,2147737007,2147737394`

---

### Another malicious binary was executed on the infected workstation from another AppData location. What was the full path to the binary?

1. Search for `index=* "C:\\Users\\Finance01\\AppData\\*\\*.exe"`. It is the first result

[SCREEN08]

**Answer:** `C:\Users\Finance01\AppData\Roaming\EasyCalc\EasyCalc.exe`

---

### What were the DLLs that were loaded from the binary from the previous question? Enter the answers in alphabetical order. (format: file1.dll,file2.dll,file3.dll)

1. Search for `index=* "C:\\Users\\Finance01\\AppData\\Roaming\\EasyCalc\\*.dll" | dedup ImageLoaded`

[SCREEN09]

**Answer:** `ffmpeg.dll,nw.dll,nw_elf.dll`
