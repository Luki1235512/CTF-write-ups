# [New Hire Old Artifacts](https://tryhackme.com/room/newhireoldartifacts)

## Investigate the intrusion attack using Splunk.

**Scenario:** You are a SOC Analyst for an MSSP (managed Security Service Provider) company called TryNotHackMe.

A newly acquired customer (Widget LLC) was recently onboarded with the managed Splunk service. The sensor is live, and all the endpoint events are now visible on TryNotHackMe's end. Widget LLC has some concerns with the endpoints in the Finance Dept, especially an endpoint for a recently hired Financial Analyst. The concern is that there was a period (December 2021) when the endpoint security product was turned off, but an official investigation was never conducted.

Your manager has tasked you to sift through the events of Widget LLC's Splunk instance to see if there is anything that the customer needs to be alerted on.

Happy Hunting!

### A Web Browser Password Viewer executed on the infected machine. What is the name of the binary? Enter the full path.

1. Search for `index=* "password viewer"`

<img width="1641" height="677" alt="SCREEN01" src="https://github.com/user-attachments/assets/f5e7f1f6-9d6e-4ed8-9fc7-e73e456ad44f" />

**Answer:** `C:\Users\FINANC~1\AppData\Local\Temp\11111.exe`

---

### What is listed as the company name?

1. The same search `index=* "password viewer"`

<img width="1639" height="677" alt="SCREEN02" src="https://github.com/user-attachments/assets/f5d87c72-c2d9-4504-98e0-02f52b6df649" />

**Answer:** `NirSoft`

---

### Another suspicious binary running from the same folder was executed on the workstation. What was the name of the binary? What is listed as its original filename? (format: file.xyz,file.xyz)

_File path should include username in long name format. https://docs.microsoft.com/en-us/windows/win32/fileio/naming-a-file_

1. Search for `index=* "C:\\Users\\Finance01\\AppData\\Local\\Temp\\*.exe" OriginalFileName="*"`

<img width="1640" height="719" alt="SCREEN03" src="https://github.com/user-attachments/assets/99662552-bf85-4a84-a732-cdd8c87beba3" />

**Answer:** `IonicLarge.exe,PalitExplorer.exe`

---

### The binary from the previous question made two outbound connections to a malicious IP address. What was the IP address? Enter the answer in a defang format.

_Cyberchef can help with defanging._

1. Search for `index=* IonicLarge.exe` and check `DestinationIp` field

<img width="886" height="436" alt="SCREEN04" src="https://github.com/user-attachments/assets/cdf6d0a1-58d2-412d-a636-3e7af454a71f" />

**Answer:** `2[.]56[.]59[.]42`

---

### The same binary made some change to a registry key. What was the key path?

1. Search for `index=* IonicLarge.exe Registry`

<img width="1628" height="801" alt="SCREEN05" src="https://github.com/user-attachments/assets/4b97e9a3-2bfe-48f1-9f8b-bcc99ebf9334" />

**Answer:** `HKLM\SOFTWARE\Policies\Microsoft\Windows Defender`

---

### Some processes were killed and the associated binaries were deleted. What were the names of the two binaries? (format: file.xyz,file.xyz)

_Process were killed with 'taskkill /im'_

1. Search for `index=* "taskkill /im"`

<img width="1629" height="759" alt="SCREEN06" src="https://github.com/user-attachments/assets/bb4000d1-a175-4e1a-b121-7402e344c7c8" />

**Answer:** `WvmIOrcfsuILdX6SNwIRmGOJ.exe,phcIAmLJMAIMSa9j9MpgJo1m.exe`

---

### The attacker ran several commands within a PowerShell session to change the behaviour of Windows Defender. What was the last command executed in the series of similar commands?

_The last command issue has the most recent time stamp_

1. Search for `index=* powershell commandline`

<img width="1639" height="680" alt="SCREEN07" src="https://github.com/user-attachments/assets/60f19d02-ad2b-4363-a0d7-fa36e74de9e8" />

**Answer:** `powershell WMIC /NAMESPACE:\\root\Microsoft\Windows\Defender PATH MSFT_MpPreference call Add ThreatIDDefaultAction_Ids=2147737394 ThreatIDDefaultAction_Actions=6 Force=True`

---

### Based on the previous answer, what were the four IDs set by the attacker? Enter the answer in order of execution. (format: 1st,2nd,3rd,4th)

1. Search for `index=* "PATH MSFT_MpPreference call Add ThreatIDDefaultAction_Ids=*"`

**Answer:** `2147735503,2147737010,2147737007,2147737394`

---

### Another malicious binary was executed on the infected workstation from another AppData location. What was the full path to the binary?

1. Search for `index=* "C:\\Users\\Finance01\\AppData\\*\\*.exe"`. It is the first result

<img width="1625" height="386" alt="SCREEN08" src="https://github.com/user-attachments/assets/7505d212-f388-45d1-b270-2a088501de76" />

**Answer:** `C:\Users\Finance01\AppData\Roaming\EasyCalc\EasyCalc.exe`

---

### What were the DLLs that were loaded from the binary from the previous question? Enter the answers in alphabetical order. (format: file1.dll,file2.dll,file3.dll)

1. Search for `index=* "C:\\Users\\Finance01\\AppData\\Roaming\\EasyCalc\\*.dll" | dedup ImageLoaded`

<img width="1629" height="624" alt="SCREEN09" src="https://github.com/user-attachments/assets/8a1ef96e-fd7f-4f3d-abf2-f044ae489372" />

**Answer:** `ffmpeg.dll,nw.dll,nw_elf.dll`
