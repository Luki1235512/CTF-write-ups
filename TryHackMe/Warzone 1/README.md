# [Warzone 1](https://tryhackme.com/room/warzoneone)

## You received an IDS/IPS alert. Time to triage the alert to determine if its a true positive.

# Your shift just started and your first network alert comes in.

You work as a Tier 1 Security Analyst L1 for a Managed Security Service Provider (MSSP). Today you're tasked with monitoring network alerts.

A few minutes into your shift, you get your first network case: **Potentially Bad Traffic** and **Malware Command and Control Activity detected**. Your race against the clock starts. Inspect the PCAP and retrieve the artifacts to confirm this alert is a true positive.

**Your tools:**

- Brim
- Network Miner
- Wireshark

### What was the alert signature for Malware Command and Control Activity Detected?

_Brim_

1. Open `Zone1.pcap` file in `Brim`

2. Search for `malware` and look for `alert.signature` column

<img width="1258" height="782" alt="SCREEN01" src="https://github.com/user-attachments/assets/dc33f854-a4f2-463e-9965-b99f7227a846" />

**Answer:** `ET Malware MirrorBlast CnC Activity M3`

---

### What is the source IP address? Enter your answer in a defanged format.

_Cyberchef can defang._

1. Same search, `src_ip` column

**Answer:** `172[.]16[.]1[.]102`

---

### What IP address was the destination IP in the alert? Enter your answer in a defanged format.

_Cyberchef can defang._

1. Same search, `dest_ip` column

**Answer:** `169[.]239[.]128[.]11`

---

### Still in VirusTotal, under Community, what threat group is attributed to this IP address?

1. Visit `https://www.virustotal.com/gui/ip-address/169.239.128.11/community` and look at `Contained in Graphs` section

<img width="1717" height="907" alt="SCREEN02" src="https://github.com/user-attachments/assets/496ba812-5ec5-4ade-80fb-1e409b751098" />

**Answer:** `TA505`

---

### What is the malware family?

1. On the same section

**Answer:** `MirrorBlast`

---

### Do a search in VirusTotal for the domain from question 4. What was the majority file type listed under Communicating Files?

_Check Relations_

1. Visit `https://www.virustotal.com/gui/ip-address/169.239.128.11/relations` and look at the `Communicating Files` section.

<img width="1717" height="907" alt="SCREEN03" src="https://github.com/user-attachments/assets/338e9bbb-8a22-4fbd-a301-9fe348ef8f7f" />

**Answer:** `Windows Installer`

---

### Inspect the web traffic for the flagged IP address; what is the user-agent in the traffic?

1. Click `Wireshark` icon in `Brim`

2. Right click on `HTTP` packet and select `Follow > HTTP Stream`

<img width="1345" height="815" alt="SCREEN04" src="https://github.com/user-attachments/assets/1c61650d-82df-4559-8bb6-b978eab700ab" />

**Answer**: `REBOL View 2.7.8.3.1`

---

### Retrace the attack; there were multiple IP addresses associated with this attack. What were two other IP addresses? Enter the IP addressed defanged and in numerical order. (format: IPADDR,IPADDR)

_Brim (HTTP logs) &amp; VT (Community tab) can help you here. Cyberchef can defang._

1. Back in `Brim` search `172.16.1.102 _path == "http"`

<img width="1256" height="784" alt="SCREEN05" src="https://github.com/user-attachments/assets/76fe23c0-f60f-4833-9c8b-43f827fc76d1" />

2. Visit:
   - `https://www.virustotal.com/gui/ip-address/142.250.74.110/community`
   - `https://www.virustotal.com/gui/ip-address/185.10.68.235/community`
   - `https://www.virustotal.com/gui/ip-address/185.183.96.147/community`
   - `https://www.virustotal.com/gui/ip-address/192.36.27.92/community`

**Answer:** `185[.]10[.]68[.]235,192[.]36[.]27[.]92`

---

### What were the file names of the downloaded files? Enter the answer in the order to the IP addresses from the previous question. (format: file.xyz,file.xyz)

_The first character in the second filename is not a lowercase or uppercase "L"._

1. Select `File Activity` in `QUERIES` section to get first file

<img width="1254" height="781" alt="SCREEN07" src="https://github.com/user-attachments/assets/425b93ba-a402-4223-912f-eba2b65213df" />

2. Search for `_path == "http" | uniq -c` to get second file

<img width="1255" height="785" alt="SCREEN06" src="https://github.com/user-attachments/assets/3fe615b0-110e-45db-a97c-60f878bd808c" />

**Answer:** `filter.msi,10opd3r_load.msi`

---

### Inspect the traffic for the first downloaded file from the previous question. Two files will be saved to the same directory. What is the full file path of the directory and the name of the two files? (format: C:\path\file.xyz,C:\path\file.xyz)

_Inspect the streams._

1. Select first file and click on the Wireshark icon

2. Click `Follow > TCP Stream`, and search for `C:\`

<img width="1131" height="753" alt="SCREEN08" src="https://github.com/user-attachments/assets/6f80f6d3-9a24-4250-905e-a07b3998cc18" />

**Answer:** `C:\ProgramData\001\arab.bin,C:\ProgramData\001\arab.exe`

---

### Now do the same and inspect the traffic from the second downloaded file. Two files will be saved to the same directory. What is the full file path of the directory and the name of the two files? (format: C:\path\file.xyz,C:\path\file.xyz)

_Inspect the streams._

1. Select second file and click on the Wireshark icon

2. Click `Follow > TCP Stream`, and search for `C:\`

<img width="971" height="681" alt="SCREEN09" src="https://github.com/user-attachments/assets/c47242c7-7219-4ff6-b082-6ff15a3dd98f" />

**Answer:** `C:\ProgramData\Local\Google\rebol-view-278-3-1.exe,C:\ProgramData\Local\Google\exemple.rb`
