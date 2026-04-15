# [Warzone 2](https://tryhackme.com/room/warzonetwo)

## You received another IDS/IPS alert. Time to triage the alert to determine if its a true positive.

# Another day, another alert.

You work as a Tier 1 Security Analyst L1 for a Managed Security Service Provider (MSSP). Again, you're tasked with monitoring network alerts.

An alert triggered: **Misc activity**, **A Network Trojan Was Detected**, and **Potential Corporate Privacy Violation**.

The case was assigned to you. Inspect the PCAP and retrieve the artifacts to confirm this alert is a true positive.

**Your tools:**

- Brim
- Network Miner
- Wireshark

### What was the alert signature for A Network Trojan was Detected?

1. Open `Brim` and search for `aler.signature`

[SCREEN01]

**Answer:** `ET MALWARE Likely Evil EXE download from MSXMLHTTP non-exe extension M2`

---

### What was the alert signature for Potential Corporate Privacy Violation?

1. The same search as above

[SCREEN02]

**Answer:** `ET POLICY PE EXE or DLL Windows file download HTTP`

---

### What was the IP to trigger either alert? Enter your answer in a defanged format.

_Cyberchef can defang._

1. The same search as above

[SCREEN03]

**Answer:** `185[.]118[.]164[.]8`

---

### Provide the full URI for the malicious downloaded file. In your answer, defang the URI.

_Cyberchef can defang._

1. Click on the `Wireshark` icon and find `HTTP` protocol

[SCREEN04]

**Answer:** `awh93dhkylps5ulnq-be[.]com/czwih/fxla[.]php?l=gap1[.]cab`

---

### What is the name of the payload within the cab file?

_Extract the file from PCAP, get the hash, then hop to VirusTotal_

1. Click `File` > `Export Objects` > `HTTP`

2. Get md5 hash

```bash
md5sum fxla.php%3fl\=gap1.cab
```

3. Check `VirusTotal`

[SCREEN05]

**Answer:** `draw.dll`

---

### What is the user-agent associated with this network traffic?

1. Back in `Wiresharh` filter for `http` and check the `Hypertext Transfer Protocol`

[SCREEN06]

**Answer:** `Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 10.0; WOW64; Trident/8.0; .NET4.0C; .NET4.0E)`

---

### What other domains do you see in the network traffic that are labelled as malicious by VirusTotal? Enter the domains defanged and in alphabetical order. (format: domain[.]zzz,domain[.]zzz)

_Check the Misc Activity alert in Brim. Cyberchef can defang._

1. Back to the `VirusTotal` in `RELATIONS` tab

2. In `Brim` search for `Misc Activity`, then `176.119.156.128 | cut id.resp_h, host | uniq -c`

[SCREEN07]

**Answer:** `a-zcorner[.]com,knockoutlights[.]com`

---

### There are IP addresses flagged as Not Suspicious Traffic. What are the IP addresses? Enter your answer in numerical order and defanged. (format: IPADDR,IPADDR)

1. Search for `"Not Suspicious Traffic"`

[SCREEN08]

**Answer:** `64[.]225[.]65[.]166,142[.]93[.]211[.]176`

---

### For the first IP address flagged as Not Suspicious Traffic. According to VirusTotal, there are several domains associated with this one IP address that was flagged as malicious. What were the domains you spotted in the network traffic associated with this IP address? Enter your answer in a defanged format. Enter your answer in alphabetical order, in a defanged format. (format: domain[.]zzz,domain[.]zzz,etc)

1. Check `RELATIONS` tab in `VirusTotal` for `64.225.65.166`

[SCREEN09]

**Answer:** `safebanktest[.]top,tocsicambar[.]xyz,ulcertification[.]xyz`

---

### Now for the second IP marked as Not Suspicious Traffic. What was the domain you spotted in the network traffic associated with this IP address? Enter your answer in a defanged format. (format: domain[.]zzz)

_Brim, Network Miner, or Wireshark_

1. Search for `142.93.211.176`

[SCREEN10]

**Answer:** `2partscow[.]top`
