# [DetectMare](https://tryhackme.com/room/detectmare)

## THM Security Services has been engaged for a Detection Engineering activity for Meridian Defense Research Institute.

# Case Briefing

**Another silent incident with no detections firing.**

You have been brought on at THM Security Services (TSS). You are a **DETECTION ENGINEER**.

Your case briefing is waiting in the TSS Operations Hub. Head to the **Active Case** tab for everything you need before you begin. The other tabs are there if you want to explore the company, the team, or previous cases.

# Active Case

DetectMare - Active investigation

| CLIENT                              | INDUSTRY   | LEAD ANALYST |
| ----------------------------------- | ---------- | ------------ |
| Meridian Defense Research Institute | Government | Tom Becker   |

## Situation Report

The TSS CSIRT team has worked an incident on a government customer, the Meridian Defense Research Institute, and generated a full report of the attack details. The activity is consistent with APT21, a China-based cyber espionage group historically focused on government, defense, and research targets. The intruder gained initial access through a spearphishing attachment opened by a propulsion-lab engineer, established a foothold using a NetTraveler-style loader, mapped the domain, harvested credentials from memory, moved laterally onto the classified file servers using stolen hashes, and staged classified weapons-program design files into encrypted archives for exfiltration.

Critically, the customer environment did not trigger a single alert during the entire attack. The activity was only surfaced after the fact through the CSIRT investigation, which means the estate is currently blind to this class of state-aligned espionage tradecraft.

## Internal Message

```
FROM: Tom Becker, Lead Detection Engineer
TO: You
SUBJECT: RE: Meridian Research Institute Incident - No Detection Rules Working
```

The CSIRT report just landed and I need you looking at it before end of day. The report is stored at the `\threat-intel` folder in the DaC app. Here's what's keeping me up: this was a full APT21 kill chain agains the Meridian Research Institute, and do you know how many alerts fired across the entire estate? Zero. Not one. No detections fired! This is a detection nightmare... or a DetectMare, as you prefer to call it.

To close those gaps fast, Jordan Blake, from the CSIRT team, has submitted several pull requests in our Detection-as-Code repository, one per step of the intrusion. So here's the job: I need you to review their PRs as soon as possible. Each PR has a Sigma rule that needs real tuning and detection engineering work, not just a tweak. Expect to rework everything if needed, and consider the institute-specific context.

Always remember that a solid research before touching the rule may define the success of your detection performance. Let's turn this DetectMare into detections that actually fire.

## Mission Parameters

1. Read and understand the attack chain at the incident report.
2. Identify the problems and why the detections from the CSIRT analyst are not working.
3. Research and understand the attacker's technique before applying the fixes.
4. Understand the customer's environment particularities before applying the fixes.
5. Tune and merge the detections in the DaC application.

## Assigned Personnel

Tom Becker - `Detection Engineering Lead`
You - `Assigned Analyst`

# Tuning Detections

Having reviewed the case details on the TSS Operations Hub, it is time to begin your detection review.

Your colleagues on the CSIRT team have collected the logs from the time of the attack and set them up in a Splunk instance connected to the TSS Detection Engineering team's Detection-as-Code (DaC) application. The application is a GitHub-style site that runs multiple checks and validations before deploying a detection rule. If you are not familiar with the DaC concept, we strongly recommend taking a look at the [AI & Automation in Detection Engineering](https://tryhackme.com/room/aiautomationdetectioneng) room, which provides you with a clear explanation of this concept.

Inside the application, you will see the `README.md` file under the `Code` tab. This file contains details on how the pipeline works and an explanation of what DaC is. The application also has an `App Instructions` button that interactively guides you.

## Lab Access

To access the Detection-as-Code interface, please follow this link:

- `https://LAB_WEB_URL.p.thmlabs.com/dac-site`

To access the Splunk instance, please follow this link:

- `https://LAB_WEB_URL.p.thmlabs.com`
- Use the following index to see the environment logs and filter for All time: `index="dac_lab"`

---

### What is the doc file opened by the infected user?

1. Start by hunting for Office applications spawning child processes. Since `WINWORD.EXE` is the Office component, and Sysmon Event Code 1 captures process creation with full parent/child lineage, this is the natural first query:

```
index="dac_lab" EventCode=1 ParentImage="*WINWORD.EXE*"
   | table _time, User, ParentCommandLine, CommandLine
```

This surfaces the full command line Word was launched with alongside whatever child process the macro spawned. Giving you both the lure document name and your first look at the malicious execution chain in one query.

**Answer:** `Hypersonic_Test_Schedule_2025.docm`

The `.docm` extension is itself a signal worth noting. It's consistent with the CSIRT report's description of a "spearphishing attachment," and the filename itself is a well-crafted, plausible lure for a propulsion-lab engineer at a defense research institute, which is exactly the kind of social-engineering specificity APT21 is known for when targeting research and government verticals.

---

### What is the PR#1 flag?

1. Fixed PR:

```
title: Spearphishing Attachment Spawns Suspicious Child Process
id: 3f9a2b10-1e44-4a2b-9b0a-1a2b3c4d5e06
status: experimental
description: Detects an Office application launching a script interpreter or living-off-the-land binary, consistent with APT21 lure documents.
author: jordan-blake
logsource:
  product: windows
  category: process_creation

detection:
  selection:
    ParentImage|endswith:
      - '\WINWORD.EXE'
      - '\EXCEL.EXE'
      - '\POWERPNT.EXE'
      - '\OUTLOOK.EXE'
    Image|endswith:
      - '\cmd.exe'
      - '\powershell.exe'
      - '\pwsh.exe'
      - '\wscript.exe'
      - '\cscript.exe'
      - '\mshta.exe'
      - '\regsvr32.exe'
      - '\rundll32.exe'
  filter_monthend_automation:
    ParentImage|endswith: '\EXCEL.EXE'
    ParentCommandLine|contains: '\Finance\MonthEnd_Template.xlsm'
    CommandLine|contains: '\ResearchIT\Automation\monthend_report.bat'
  condition: selection and not filter_monthend_automation
```

[SCREEN01]

---

### What is the filename of the internal tool that may cause false positives in PR#2 if not properly filtered?

1. PR #2 concerns signed-binary proxy execution. A technique where trusted Windows binaries like `rundll32.exe` are abused to run malicious code while appearing legitimate. To find what _legitimately_ triggers this pattern in Meridian's environment, pull every `rundll32.exe` execution across the estate and inspect the parent processes and command lines for a recognizable internal deployment or licensing tool:

```
index="dac_lab" EventCode=1 Image="*rundll32.exe*"
| table _time, User, ComputerName, ParentImage, CommandLine
| sort _time
```

**Answer:** `researchdeploy.exe`

This is Meridian's internal software-deployment agent, and it apparently uses `rundll32.exe` as part of its normal package-installation workflow - staging .dat package files under `\ProgramData\ResearchIT\pkg\`. That legitimate behavior looks structurally almost identical to the NetTraveler dropper's staged-path-plus-suspicious-extension pattern, which is precisely why it needs a carefully scoped exclusion rather than a blanket path-based one.

---

### What is the PR#2 flag?

1. Fixed PR:

```
title: Signed Binary Proxy Execution of NetTraveler Dropper
id: 3f9a2b10-1e44-4a2b-9b0a-1a2b3c4d5e07
status: experimental
description: Detects APT21 using signed Windows binaries to proxy execute the NetTraveler loader from a non-standard path.
author: jordan-blake
logsource:
  product: windows
  category: process_creation

detection:
  selection_proxy:
    Image|endswith:
      - '\regsvr32.exe'
      - '\rundll32.exe'
      - '\mshta.exe'
  selection_remote:
    CommandLine|contains: 'http'
  selection_staged_path:
    CommandLine|contains:
      - '\AppData\Roaming\'
      - '\AppData\Local\Temp\'
      - '\ProgramData\'
  selection_staged_ext:
    CommandLine|contains:
      - '.dat'
      - '.tmp'
      - '.sct'
  filter_researchdeploy:
    ParentImage|endswith: '\Deploy\researchdeploy.exe'
    CommandLine|contains: '\ProgramData\ResearchIT\pkg\'
    CommandLine|contains: '.dat'
  filter_solidworks_license:
    ParentImage|endswith: '\SOLIDWORKS\sldworks.exe'
    CommandLine|contains: '\AppData\Local\Temp\'
    CommandLine|endswith: 'LicenseCheck_CAD.dat'
  condition: >
    selection_proxy and
    (selection_remote or (selection_staged_path and selection_staged_ext))
    and not (filter_researchdeploy or filter_solidworks_license)
```

[SCREEN02]

---

### What is the username that the attacker used to execute the LSASS dump?

1. Credential access techniques targeting LSASS are best hunted via Sysmon's process-access telemetry, since that captures which process opened a handle to `lsass.exe` and with what access rights:

```
index="dac_lab" sourcetype="sysmon:process_access" TargetImage="*lsass.exe*"
| table _time, User, ComputerName, SourceImage, TargetImage, GrantedAccess
| sort _time
```

2. Once you have a candidate timestamp and host from the process-access event, pivot to process-creation logs on that same host in a tight time window around the access event, to confirm what tool/technique was actually used and under which account context it ran:

```
index="dac_lab" sourcetype="sysmon:process_creation" ComputerName="RESEARCH-ENG14"
earliest="03/11/2025:10:00:00" latest="03/11/2025:10:40:00"
| table _time, User, ParentImage, Image, CommandLine
| sort _time
```

**Answer:** `m.okafor`

This confirms the account used to execute the LSASS access was a compromised user account, which lines up with the CSIRT narrative that the actor "harvested credentials from memory" after establishing an initial foothold. This is the pivot point in the kill chain between execution/persistence and lateral movement.

---

### What is the PR#3 flag?

1. Fixed PR:

```
title: LSASS Memory Access for Credential Theft
id: 3f9a2b10-1e44-4a2b-9b0a-1a2b3c4d5e09
status: experimental
description: Detects suspicious handle access to LSASS consistent with APT21 credential dumping.
author: jordan-blake
logsource:
  product: windows
  category: process_access

detection:
  selection_target:
    TargetImage|endswith: '\lsass.exe'
  selection_module:
    CallTrace|contains:
      - 'dbghelp.dll'
      - 'dbgcore.dll'
      - 'comsvcs.dll'
  selection_unknown:
    CallTrace|contains: 'UNKNOWN'
  filter_werfault:
    SourceImage|endswith: '\WerFault.exe'
    CallTrace|contains: 'dbghelp.dll'
    GrantedAccess: '0x1410'
  filter_vaultagent:
    SourceImage|endswith: '\ResearchPAM\vaultagent.exe'
    CallTrace|contains: 'UNKNOWN'
    GrantedAccess: '0x1438'
  condition: >
    selection_target and (selection_module or selection_unknown)
    and not (filter_werfault or filter_vaultagent)
```

[SCREEN03]

---

### When did the pass-the-hash authentication happen? Answer Format: The `timestamp` from the first line of the raw event in Splunk, in MM/DD/YYYY HH:MM:SS.mmm AM/PM. For example `06/22/2025 09:35:00.000 AM`

1. Start broad to understand what Windows Security event types are present in the dataset at all, so you know which EventCode corresponds to logon events:

```
index="dac_lab" sourcetype="wineventlog:security"
| stats count by EventCode
```

2. Narrow to EventCode 4624 across the date of the incident to look for the lateral-movement logon:

```
index="dac_lab" sourcetype="wineventlog:security" EventCode=4624
earliest="03/11/2025:00:00:00" latest="03/12/2025:00:00:00"
| table _time, ComputerName, _raw
| sort _time
```

3. Narrow further to the classified file server specifically, in a tight window around the timeframe suggested by the earlier LSASS-dump timeline, to isolate the exact pass-the-hash logon event and pull its raw event text for the precise timestamp:

```
index="dac_lab" sourcetype="wineventlog:security" EventCode=4624
ComputerName="FS-CLASSIFIED01.research.local"
earliest="03/11/2025:10:35:00" latest="03/11/2025:10:45:00"
| table _time, _raw
```

**Answer:** `3/11/2025 10:40:00.000 AM`

This timestamp sits roughly in line with the credential-theft activity under `m.okafor`, confirming the expected kill-chain sequencing: LSASS dump -> hash theft -> pass-the-hash logon to the classified file server, all within the same operational window.

---

### What is the PR#4 flag?

1. Fixed PR:

```
title: Pass the Hash Lateral Movement to Classified Host
id: 3f9a2b10-1e44-4a2b-9b0a-1a2b3c4d5e0a
status: experimental
description: Detects APT21 NTLM pass-the-hash logon followed by remote service creation.
author: jordan-blake
logsource:
  product: windows
  service: security

detection:
  selection_logon:
    EventID: 4624
    LogonType: 3
    AuthenticationPackageName: 'NTLM'
  filter_svc_mes:
    TargetUserName: 'svc_mes'
    WorkstationName: 'MES-LEGACY01'
  filter_svc_cluster:
    TargetUserName: 'svc_cluster'
  filter_machine_accounts:
    TargetUserName|endswith: '$'
  selection_service_path:
    EventID: 7045
    ServiceFileName|contains: '\ProgramData\'
  selection_service_lolbin:
    EventID: 7045
    ServiceFileName|contains:
      - 'cmd.exe'
      - 'powershell'
      - 'pwsh'
      - '-enc'
      - '-EncodedCommand'
  filter_patchdeploy:
    ServiceFileName|contains: '\ResearchIT\Patch\apply.ps1'
  condition: >
    (selection_logon and not (filter_svc_mes or filter_svc_cluster or filter_machine_accounts))
    or selection_service_path
    or (selection_service_lolbin and not filter_patchdeploy)
```

[SCREEN04]

---

### In which folder should an attacker place a malicious binary to make it look like a legitimate backup routine?

1. The final stage of the kill chain is staging and archiving classified files, so hunting for archive-utility execution reveals both the legitimate backup pattern and the disguise the attacker used to blend in with it:

```
index="dac_lab" EventCode=1 Image="*7z.exe*"
| table _time, User, ComputerName, ParentImage, CommandLine
| sort _time
```

**Answer:** `D:\Backups\nightly`

---

### What is the PR#5 flag?

1. Fixed PR:

```
title: Weapons Program Data Staged and Archived for Exfiltration
id: 3f9a2b10-1e44-4a2b-9b0a-1a2b3c4d5e0b
status: experimental
description: Detects APT21 compressing classified design documents into a password-protected archive prior to exfiltration.
author: jordan-blake
logsource:
  product: windows
  category: process_creation

detection:
  selection_archive_syntax:
    CommandLine|contains:
      - ' a -mx'
      - '-hp'
      - '-p'
  selection_powershell_archive:
    CommandLine|contains: 'Compress-Archive'
  selection_target:
    CommandLine|contains:
      - 'fs-classified'
      - '.sldprt'
      - '.catpart'
      - '.dwg'
  filter_researchbackup:
    ParentImage|endswith: '\ResearchBackup\researchbackup.exe'
    CommandLine|contains: 'D:\Backups\nightly\'
  filter_user_photos:
    ParentImage|endswith: '\explorer.exe'
    CommandLine|contains: '\Pictures\*.jpg'
  filter_solidworks_autobackup:
    ParentImage|endswith: '\SOLIDWORKS\sldworks.exe'
    CommandLine|contains: '\SOLIDWORKS\SW_AutoBackup\'
  condition: >
    ((selection_archive_syntax or selection_powershell_archive) and selection_target)
    and not (filter_researchbackup or filter_user_photos or filter_solidworks_autobackup)
```

[SCREEN05]
