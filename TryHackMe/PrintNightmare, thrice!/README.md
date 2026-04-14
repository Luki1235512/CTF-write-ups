# [PrintNightmare, thrice!](https://tryhackme.com/room/printnightmarec3kj)

## The nightmare continues.. Search the artifacts on the endpoint, again, to determine if the employee used any of the Windows Printer Spooler vulnerabilities to elevate their privileges.

# Detection

**Scenario:** After discovering the PrintNightmare attack the security team pushed an emergency patch to all the endpoints. The PrintNightmare exploit used previously no longer works. All is well. Unfortunately, the same 2 employees discovered yet another exploit that can possibly work on a fully patched endpoint to elevate their privileges.

**Task:** Inspect the artifacts on the endpoint to detect the PrintNightmare exploit used.

### What remote address did the employee navigate to?

_Check the SMB traffic in the pcap file_

1. Open `traffic.pcap` in `Wireshark` and search for `smb` ...

<img width="750" height="186" alt="SCREEN01" src="https://github.com/user-attachments/assets/b675c690-4c05-407d-8f32-6ae65e797654" />

**Answer:** `20.188.56.147`

---

### Per the PCAP, which user returns a STATUS_LOGON_FAILURE error?

_Wireshark_

1. Press `ctrl + F` select `String` and search for `STATUS_LOGON_FAILURE`, then select `SMB2` > `SMB2 Header` > `Session Id`

<img width="1400" height="589" alt="SCREEN02" src="https://github.com/user-attachments/assets/e303dc95-492f-49a3-a444-86e10cd9f6b4" />

**Answer:** `THM-PRINTNIGHT0\rjones`

---

### Which user successfully connects to an SMB share?

_Wireshark_

1. Use display filter: `smb2 && smb2.nt_status == 0x0` look for packets such as `Session Setup Response`, then select `SMB2` > `SMB2 Header` > `Session Id`

<img width="1401" height="588" alt="SCREEN03" src="https://github.com/user-attachments/assets/321fad2f-4f78-4f8a-9cdd-c784ba6708da" />

**Answer:** `THM-PRINTNIGHT0/gentilguest`

---

### What is the first remote SMB share the endpoint connected to? What was the first filename? What was the second? (format: answer,answer,answer)

_Wireshark or Brim_

1. Use the display filter `smb2 && smb2.cmd == 0x03` the first share is in `Info` column

<img width="1399" height="182" alt="SCREEN04" src="https://github.com/user-attachments/assets/83234156-a935-483f-b5fa-a8ba4499743e" />

2. Use the display filter `smb2 && smb2.cmd == 0x05` the first two files are in `Info` column

<img width="1400" height="579" alt="SCREEN05" src="https://github.com/user-attachments/assets/03d37009-c03d-41a3-aa2b-6579b96df573" />

**Answer:** `\\printnightmare.gentilkiwi.com\IPC$,srvsvc,spoolss`

---

### From which remote SMB share was malicious DLL obtained? What was the path to the remote folder for the first DLL? How about the second? (format: answer,answer,answer)

_Many DLLs were downloaded but one stands out in association to a hack tool. Brim will be the most useful._

1. In the query bar search for `_path=~smb* OR _path=dce_rpc`

<img width="1402" height="733" alt="SCREEN06" src="https://github.com/user-attachments/assets/d716fdad-1add-4f67-a25e-3662ba75591e" />

**Answer:** `\\printnightmare.gentilkiwi.com\print$,\x64\3\mimispool.dll,\W32X86\3\mimispool.dll`

---

### What was the first location the malicious DLL was downloaded to on the endpoint? What was the second?

_Check the event logs_

1. Search for `mimispool.dll`

<img width="1037" height="610" alt="SCREEN07" src="https://github.com/user-attachments/assets/9a9a2b43-ed2f-45db-b025-cec232707e3f" />

<img width="1038" height="609" alt="SCREEN08" src="https://github.com/user-attachments/assets/c260c435-2b5e-4114-81d8-0e4a1c09c6a6" />

**Answer:** `C:\Windows\system32\spool\drivers\X64\3,C:\Windows\system32\spool\drivers\W32X86\3`

---

### What is the folder that has the name of the remote printer server the user connected to? (provide the full folder path)

1. Search for `printnightmare.gentilkiwi.com` in file explorer

<img width="1124" height="597" alt="SCREEN09" src="https://github.com/user-attachments/assets/2b82f1ab-30e9-4b85-b852-c6550117e819" />

**Answer:** `C:\Windows\System32\spool\SERVERS\printnightmare.gentilkiwi.com`

---

### What is the name of the printer the DLL added?

_Check Microsoft-Windows-PrintService_

1. Search for `\\printnightmare.gentilkiwi.com`

<img width="1037" height="608" alt="SCREEN10" src="https://github.com/user-attachments/assets/ce160777-7dd1-41e2-8457-79fc695841eb" />

**Answer:** `Kiwi Legit Printer`

---

### What was the process ID for the elevated command prompt? What was its parent process? (format: answer,answer)

1. In Process Monitor filter by `cmd.exe`. This showed the start of the cmd process with its PID and Parent PID. I then search for the Parent PID to identify the parent process name.

**Answer:** `5408, spoolsv.exe`

---

### What command did the user perform to elevate privileges?

1. In Process Monitor filter by process name `cmd.exe` and the operation `Process Create`. This displays all commands run from `cmd.exe`, revealing the privilege escalation command.

**Answer:** `net localgroup administrators rjones /add`
