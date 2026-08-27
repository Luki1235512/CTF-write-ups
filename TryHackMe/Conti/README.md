# [Conti](https://tryhackme.com/room/contiransomwarehgh)

## An Exchange server was compromised with ransomware. Use Splunk to investigate how the attackers compromised the server.

# SITREP

Some employees from your company reported that they can’t log into Outlook. The Exchange system admin also reported that he can’t log in to the Exchange Admin Center. After initial triage, they discovered some weird readme files settled on the Exchange server.

Read the latest on the Conti ransomware [here](https://www.bleepingcomputer.com/news/security/fbi-cisa-and-nsa-warn-of-escalating-conti-ransomware-attacks/).

**Task:** You are assigned to investigate this situation. Use Splunk to answer the questions below regarding the Conti ransomware.

### Can you identify the location of the ransomware?

_Hint: Look for a common Windows binary located in an unusual location._

1. Ransomware payloads are frequently disguised as, or renamed to look like, legitimate Windows binaries but are dropped into non-standard, user-writable locations rather than `C:\Windows\System32`. The fastest way to spot this is to pull every file-creation event from Sysmon and scan for a well-known binary name sitting somewhere it shouldn't be.

```
index=main sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=11
| table _time, Image, TargetFilename
| sort _time
```

**Results:**

```
2021-09-08 12:59:08	C:\Windows\system32\wbem\unsecapp.exe	C:\Users\Administrator\Documents\cmd.exe
```

A `cmd.exe` sitting under `C:\Users\Administrator\Documents\` rather than `C:\Windows\System32\` is the giveaway. Legitimate Windows binaries don't get copied into a user's Documents folder.

**Answer:** `C:\Users\Administrator\Documents\cmd.exe`

---

### What is the Sysmon event ID for the related file creation event?

1. Having identified the suspicious file path in Q1, filter directly on that `TargetFilename` to confirm which Sysmon Event Code logged its creation. This is straightforward validation. Sysmon's `FileCreate` event is always Event Code 11, but the question wants you to demonstrate that with evidence from the actual log entry.

```
index=main sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=11 TargetFilename="*Documents\\cmd.exe"
| table _time, EventCode, Image, TargetFilename
```

**Results:**

```
2021-09-08 12:59:08	11	C:\Windows\system32\wbem\unsecapp.exe	C:\Users\Administrator\Documents\cmd.exe
```

**Answer:** `11`

---

### Can you find the MD5 hash of the ransomware?

1. File creation events don't include hashes. for that you need the corresponding process execution event, which logs `Hashes` when the dropped binary is actually run. Pivot the search from the file path to the `Image` field and look for a matching process-launch event with hash data attached.

```
index=main sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1 Image="*Documents\\cmd.exe"
| table _time, Image, ParentImage, CommandLine, Hashes
```

**Results:**

```
2021-09-08 13:05:32	C:\Users\Administrator\Documents\cmd.exe	C:\Windows\System32\cmd.exe	cmd.exe	MD5=290C7DFB01E50CEA9E19DA81A781AF2C,SHA256=53B1C1B2F41A7FC300E97D036E57539453FF82001DD3F6ABF07F4896B1F9CA22,IMPHASH=23F815785DB238377F4513BE54DBA574
```

Submitting `290C7DFB01E50CEA9E19DA81A781AF2C` to VirusTotal confirms multiple AV vendors flag it as ransomware.

**Answer:** `290c7dfb01e50cea9e19da81a781af2c`

---

### What file was saved to multiple folder locations?

1. Conti drops a copy of its ransom note into every folder it encrypts, so the note file name will appear dozens of times across many different `TargetFilename` paths, all created by the same parent process within a tight time window. Re-running the Event Code 11 search and looking at the pattern of repeated filenames reveals this.

```
index=main sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=11
| table _time, Image, TargetFilename
| sort _time
```

**Results:**

```
2021-09-08 13:05:45	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\readme.txt
2021-09-08 13:06:57	c:\Users\Administrator\Documents\cmd.exe	C:\Users\.NET v4.5 Classic\Downloads\readme.txt
2021-09-08 13:06:57	c:\Users\Administrator\Documents\cmd.exe	C:\Users\.NET v4.5\Downloads\readme.txt
2021-09-08 13:06:58	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Administrator\Downloads\readme.txt
2021-09-08 13:08:22	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Administrator.BELLYBEAR\Downloads\readme.txt
2021-09-08 13:08:23	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Public\Downloads\readme.txt
2021-09-08 13:08:23	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\Videos\readme.txt
2021-09-08 13:08:23	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\Saved Games\readme.txt
2021-09-08 13:08:23	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\Pictures\readme.txt
2021-09-08 13:08:23	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\Music\readme.txt
2021-09-08 13:08:23	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\Links\readme.txt
2021-09-08 13:08:23	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\Favorites\readme.txt
2021-09-08 13:08:23	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\Downloads\readme.txt
2021-09-08 13:08:23	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\Documents\readme.txt
2021-09-08 13:08:23	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\Desktop\readme.txt
2021-09-08 13:08:23	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\AppData\readme.txt
2021-09-08 13:08:34	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\AppData\Roaming\readme.txt
2021-09-08 13:08:34	c:\Users\Administrator\Documents\cmd.exe	C:\Users\Default\AppData\Local\readme.txt
```

Eighteen `readme.txt` drops across every profile folder inside a roughly three-minute window is the fingerprint of an automated ransom-note-dropping routine.

**Answer:** `readme.txt`

---

### What was the command the attacker used to add a new user to the compromised system?

1. Attackers commonly create a local backdoor account for persistence, in case their initial foothold gets cleaned up. This shows up as a Sysmon `ProcessCreate` event where `net.exe`/`net1.exe` is invoked with user and add in the command line. Searching Event Code 1 for that pattern surfaces it directly.

```
index=main sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
| search CommandLine="*net*user*" CommandLine="*add*"
| table _time, ParentImage, Image, CommandLine, User
| sort _time
```

**Results:**

```
2021-09-08 13:04:10	C:\Windows\System32\net.exe	C:\Windows\System32\net1.exe	C:\Windows\system32\net1  user /add securityninja hardToHack123$
NOT_TRANSLATED
NT AUTHORITY\SYSTEM
```

The command creates a new local user named `securityninja` with the password `hardToHack123$`, run in the context of `NT AUTHORITY\SYSTEM`.

**Answer:** `net user /add securityninja hardToHack123$`

---

### The attacker migrated the process for better persistence. What is the migrated process image (executable), and what is the original process image (executable) when the attacker got on the system?

_Hint: Try Sysmon event code 8._

1. Sysmon Event Code 8 logs when one process injects a thread into another. The technique behind Meterpreter/Cobalt Strike "process migration," where an attacker moves their implant from an initial, possibly noisy or short-lived process into a longer-lived, more innocuous-looking one for stealthier persistence. `SourceImage` is the process doing the injecting; `TargetImage` is the process being migrated into.

```
index=main sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=8
| table _time, SourceImage, TargetImage, StartAddress
| sort _time
```

**Results:**

```
2021-09-08 12:54:12	C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe	C:\Windows\System32\wbem\unsecapp.exe	0x000001BFEE130000
```

`powershell.exe` consistent with the attacker's initial code execution via the web shell running a PowerShell payload injects into `unsecapp.exe`, a legitimate-looking WMI component that blends into normal Exchange/WMI activity.

**Answer:** `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe,C:\Windows\System32\wbem\unsecapp.exe`

---

### The attacker also retrieved the system hashes. What is the process image used for getting the system hashes?

_Hint: Try Sysmon event code 8 & check Target Image._

1. Dumping credential hashes from memory requires reading LSASS's memory space, which shows up as another `CreateRemoteThread` event. This time with `lsass.exe` as the `TargetImage`. Filtering the same Event Code 8 dataset for that target isolates the credential-access step.

```
index=main sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=8
| table _time, SourceImage, TargetImage, StartAddress
| sort _time
```

**Results:**

```
2021-09-08 12:55:30	C:\Windows\System32\wbem\unsecapp.exe	C:\Windows\System32\lsass.exe	0x000001D471950000
```

This confirms the attack chain: PowerShell -> migrated into `unsecapp.exe` -> `unsecapp.exe` then injects into `lsass.exe` about a minute later to harvest credential material from memory, which is precisely how the attacker was later able to create the `securityninja` backdoor account and move around the environment with elevated rights.

**Answer:** `C:\Windows\System32\lsass.exe`

---

### What is the web shell the exploit deployed to the system?

_Hint: Try looking in the IIS logs for POST requests._

1. Everything traced so far had to start somewhere, and since this is an internet-facing Exchange server, the initial access vector is almost certainly a web-based exploit against OWA/ECP. IIS access logs record every HTTP request; POST requests are the interesting ones here since they represent an attacker sending something to the server rather than just browsing. Aggregating POST requests by URI and sorting by count highlights both normal high-volume Exchange traffic and low-count, unusual-looking paths.

```
index=main sourcetype=iis "POST"
| rex field=_raw "^\S+\s\S+\s\S+\s(?<method>\S+)\s(?<uri>\S+)"
| where method="POST"
| stats count by uri
| sort -count
```

**Results:**

```
/powershell	216
/mapi/emsmdb/	208
/ecp/DDI/DDIService.svc/GetList	177
/OWA/auth.owa	101
/owa/service.svc	67
/Autodiscover/autodiscover.json	29
/Microsoft-Server-ActiveSync/default.eas	27
/owa/ev.owa2	15
/owa/auth.owa	8
/owa/auth/i3gfPctK1c2x.aspx	4
/owa/plt1.ashx	3
/owa/sessiondata.ashx	3
/ecp/DDI/DDIService.svc/NewObject	2
/ecp/DDI/DDIService.svc/GetObject	1
/owa/lang.owa	1
```

Everything above the fold is legitimate Exchange/OWA traffic that any mail server generates constantly. `/owa/auth/i3gfPctK1c2x.aspx`, however, is a random, non-default, 16-character-name `.aspx` file sitting inside the OWA authentication directory. A directory that should only ever contain Microsoft's own logon assets, never a user-uploaded page.

**Answer:** `i3gfPctK1c2x.aspx`

---

### What is the command line that executed this web shell?

_Hint: Check the CommandLine._

1. With the web shell's filename identified, search Sysmon `ProcessCreate` events for any command line referencing it. This should reveal how the attacker prepared the file to be writable/executable after it was dropped by the exploit.

```
index=main sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" "i3gfPctK1c2x.aspx"
| table _time, ParentImage, Image, CommandLine
```

**Results:**

```
2021-09-08 12:52:09	C:\Windows\System32\cmd.exe	C:\Windows\System32\attrib.exe	attrib.exe  -r \\\\win-aoqkg2as2q7.bellybear.local\C$\Program Files\Microsoft\Exchange Server\V15\FrontEnd\HttpProxy\owa\auth\i3gfPctK1c2x.aspx
```

`attrib.exe -r` clears the read-only flag on the web shell file, letting the attacker write to it going forward. This is timestamped at **12:52:09**, a couple of minutes before the PowerShell -> process-migration activity, which lines up: the web shell is dropped and unlocked first, then used to launch the PowerShell payload that pivots into `unsecapp.exe` and, from there, `lsass.exe`.

**Answer:** `attrib.exe  -r \\\\win-aoqkg2as2q7.bellybear.local\C$\Program Files\Microsoft\Exchange Server\V15\FrontEnd\HttpProxy\owa\auth\i3gfPctK1c2x.aspx`

---

### What three CVEs did this exploit leverage? Provide the answer in ascending order.

_Hint: External research required._

1. This question can't be answered from the Splunk data alone. It requires cross-referencing the observed TTPs against public threat intelligence on Conti's exploitation of Microsoft Exchange servers. Publicly documented Conti campaigns from mid-2021 onward are associated with the following three CVEs as part of the group's known exploited-vulnerability set for initial access to Exchange/edge infrastructure.

**Answer:** `CVE-2018-13374,CVE-2018-13379,CVE-2020-0796`
