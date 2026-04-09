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

[SCREEN01]

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

[SCREEN02]

**Answer:** `TA505`

---

### What is the malware family?

1. On the same section

**Answer:** `MirrorBlast`

---

### Do a search in VirusTotal for the domain from question 4. What was the majority file type listed under Communicating Files?

_Check Relations_

1. Visit `https://www.virustotal.com/gui/ip-address/169.239.128.11/relations` and look at the `Communicating Files` section.

[SCREEN03]

**Answer:** `Windows Installer`

---

### Inspect the web traffic for the flagged IP address; what is the user-agent in the traffic?

1. Click `Wireshark` icon in `Brim`

2. Right click on `HTTP` packet and select `Follow > HTTP Stream`

[SCREEN04]

**Answer**: `REBOL View 2.7.8.3.1`

---

### Retrace the attack; there were multiple IP addresses associated with this attack. What were two other IP addresses? Enter the IP addressed defanged and in numerical order. (format: IPADDR,IPADDR)

_Brim (HTTP logs) &amp; VT (Community tab) can help you here. Cyberchef can defang._

1. Back in `Brim` search `172.16.1.102 _path == "http"`

[SCREEN05]

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

[SCREEN07]

2. Search for `_path == "http" | uniq -c` to get second file

[SCREEN06]

**Answer:** `filter.msi,10opd3r_load.msi`

---

### Inspect the traffic for the first downloaded file from the previous question. Two files will be saved to the same directory. What is the full file path of the directory and the name of the two files? (format: C:\path\file.xyz,C:\path\file.xyz)

_Inspect the streams._

1. Select first file and click on the Wireshark icon

2. Click `Follow > TCP Stream`, and search for `C:\`

[SCREEN08]

**Answer:** `C:\ProgramData\001\arab.bin,C:\ProgramData\001\arab.exe`

---

### Now do the same and inspect the traffic from the second downloaded file. Two files will be saved to the same directory. What is the full file path of the directory and the name of the two files? (format: C:\path\file.xyz,C:\path\file.xyz)

_Inspect the streams._

1. Select second file and click on the Wireshark icon

2. Click `Follow > TCP Stream`, and search for `C:\`

[SCREEN09]

**Answer:** `C:\ProgramData\Local\Google\rebol-view-278-3-1.exe,C:\ProgramData\Local\Google\exemple.rb`
