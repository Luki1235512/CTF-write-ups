# [PS Eclipse](https://tryhackme.com/room/posheclipse)

## Use Splunk to investigate the ransomware activity.

# Ransomware or not

**Scenario:** You are a SOC Analyst for an MSSP (Managed Security Service Provider) company called **TryNotHackMe**.

A customer sent an email asking for an analyst to investigate the events that occurred on Keegan's machine on **Monday, May 16th, 2022**. The client noted that **the machine** is operational, but some files have a weird file extension. The client is worried that there was a ransomware attempt on Keegan's device.

Your manager has tasked you to check the events in Splunk to determine what occurred in Keegan's device.

Happy Hunting!

### A suspicious binary was downloaded to the endpoint. What was the name of the binary?

1. Query `index=* DestinationIp="3.17.7.232"`. It is the first event

[SCREEN01]

**Answer:** `OUTSTANDING_GUTTER.exe`

---

### What is the address the binary was downloaded from? Add http:// to your answer & defang the URL.

_Cyberchef can help with defanging the URL._

1. Query `index=* powershell CommandLine="*"`

[SCREEN02]

2. Use [CyberChef](https://gchq.github.io/CyberChef/) `From Base64` > `Decode text UTF-16LE(1200)` > `Extract URLs` > `Defang URL` ...

[SCREEN03]

**Answer:** `hxxp[://]886e-181-215-214-32[.]ngrok[.]io`

---

### What Windows executable was used to download the suspicious binary? Enter full path.

1. Query `index=* 886e-181-215-214-32.ngrok.io`

[SCREEN04]

**Answer:** `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`

---

### What command was executed to configure the suspicious binary to run with elevated privileges?

_Event Code 12 will help here. Note that the attacker tried multiple attempts to configure this command correctly_

1. Query `index=* powershell.exe`. It is the top `CommandLine`

[SCREEN05]

---

### What permissions will the suspicious binary run as? What was the command to run the binary with elevated privileges? (Format: `User` + `;` + `CommandLine`)

1. Query `index=* CommandLine="*" "OUTSTANDING_GUTTER.exe"`

[SCREEN06]

[SCREEN07]

**Answer:**: `NT AUTHORITY\SYSTEM;"C:\Windows\system32\schtasks.exe" /Run /TN OUTSTANDING_GUTTER.exe`

---

### The suspicious binary connected to a remote server. What address did it connect to? Add http:// to your answer & defang the URL.

_Cyberchef can help with defanging the URL._

1. Query `index=* "OUTSTANDING_GUTTER.exe"` and check the `QueryName`

[SCREEN08]

**Answer:**: `hxxp[://]9030-181-215-214-32[.]ngrok[.]io`

---

### A PowerShell script was downloaded to the same location as the suspicious binary. What was the name of the file?

1. Query `index=* .ps1`

[SCREEN09]

**Answer:**: `script.ps1`

---

### The malicious script was flagged as malicious. What do you think was the actual name of the malicious script?

_Check VirusTotal for the hash of the PowerShell script._

1. Copy any hash from `Hashes` field and search on [VirusTotal](https://www.virustotal.com/gui/file/e5429f2e44990b3d4e249c566fbf19741e671c0e40b809f87248d9ec9114bef9)

[SCREEN10]

**Answer:** `BlackSun.ps1`

---

### A ransomware note was saved to disk, which can serve as an IOC. What is the full path to which the ransom note was saved?

1. Query `index=* .txt`

[SCREEN11]

**Answer:** `C:\Users\keegan\Downloads\vasg6b0wmw029hd\BlackSun_README.txt`

---

### The script saved an image file to disk to replace the user's desktop wallpaper, which can also serve as an IOC. What is the full path of the image?

1. Query `index=* blacksun`

[SCREEN12]

**Answer:** `C:\Users\Public\Pictures\blacksun.jpg`
