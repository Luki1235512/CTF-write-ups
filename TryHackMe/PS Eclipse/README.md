# [PS Eclipse](https://tryhackme.com/room/posheclipse)

## Use Splunk to investigate the ransomware activity.

# Ransomware or not

**Scenario:** You are a SOC Analyst for an MSSP (Managed Security Service Provider) company called **TryNotHackMe**.

A customer sent an email asking for an analyst to investigate the events that occurred on Keegan's machine on **Monday, May 16th, 2022**. The client noted that **the machine** is operational, but some files have a weird file extension. The client is worried that there was a ransomware attempt on Keegan's device.

Your manager has tasked you to check the events in Splunk to determine what occurred in Keegan's device.

Happy Hunting!

### A suspicious binary was downloaded to the endpoint. What was the name of the binary?

1. Query `index=* DestinationIp="3.17.7.232"`. It is the first event

<img width="1641" height="714" alt="SCREEN01" src="https://github.com/user-attachments/assets/19271e97-ca57-47c4-9be9-7a4a3e4fd037" />

**Answer:** `OUTSTANDING_GUTTER.exe`

---

### What is the address the binary was downloaded from? Add http:// to your answer & defang the URL.

_Cyberchef can help with defanging the URL._

1. Query `index=* powershell CommandLine="*"`

<img width="1637" height="746" alt="SCREEN02" src="https://github.com/user-attachments/assets/b54ed6f1-e3a6-46a0-b710-501e721ca98f" />

2. Use [CyberChef](https://gchq.github.io/CyberChef/) `From Base64` > `Decode text UTF-16LE(1200)` > `Extract URLs` > `Defang URL` ...

<img width="1715" height="791" alt="SCREEN03" src="https://github.com/user-attachments/assets/bc04f3fd-f8a2-486a-962c-ec678d38704c" />

**Answer:** `hxxp[://]886e-181-215-214-32[.]ngrok[.]io`

---

### What Windows executable was used to download the suspicious binary? Enter full path.

1. Query `index=* 886e-181-215-214-32.ngrok.io`

<img width="1635" height="570" alt="SCREEN04" src="https://github.com/user-attachments/assets/5425ed15-132f-41ab-9520-a8b3fed1822d" />

**Answer:** `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`

---

### What command was executed to configure the suspicious binary to run with elevated privileges?

_Event Code 12 will help here. Note that the attacker tried multiple attempts to configure this command correctly_

1. Query `index=* powershell.exe`. It is the top `CommandLine`

<img width="888" height="802" alt="SCREEN05" src="https://github.com/user-attachments/assets/d68c5f73-019b-4a5e-8504-ae072dbfe618" />

---

### What permissions will the suspicious binary run as? What was the command to run the binary with elevated privileges? (Format: `User` + `;` + `CommandLine`)

1. Query `index=* CommandLine="*" "OUTSTANDING_GUTTER.exe"`

<img width="1641" height="715" alt="SCREEN06" src="https://github.com/user-attachments/assets/e7b2f95d-5abd-46df-9927-34a18b7a4a1b" />

<img width="1627" height="765" alt="SCREEN07" src="https://github.com/user-attachments/assets/59b02dae-fea5-43cd-8155-a780cc2a80bf" />

**Answer:** `NT AUTHORITY\SYSTEM;"C:\Windows\system32\schtasks.exe" /Run /TN OUTSTANDING_GUTTER.exe`

---

### The suspicious binary connected to a remote server. What address did it connect to? Add http:// to your answer & defang the URL.

_Cyberchef can help with defanging the URL._

1. Query `index=* "OUTSTANDING_GUTTER.exe"` and check the `QueryName`

<img width="879" height="640" alt="SCREEN08" src="https://github.com/user-attachments/assets/f9f87069-3390-4573-913b-26a1ad8c8ea4" />

**Answer:** `hxxp[://]9030-181-215-214-32[.]ngrok[.]io`

---

### A PowerShell script was downloaded to the same location as the suspicious binary. What was the name of the file?

1. Query `index=* .ps1`

<img width="1641" height="592" alt="SCREEN09" src="https://github.com/user-attachments/assets/5cf794cf-474b-42fb-8c94-f076eb6ffeaa" />

**Answer:** `script.ps1`

---

### The malicious script was flagged as malicious. What do you think was the actual name of the malicious script?

_Check VirusTotal for the hash of the PowerShell script._

1. Copy any hash from `Hashes` field and search on [VirusTotal](https://www.virustotal.com/gui/file/e5429f2e44990b3d4e249c566fbf19741e671c0e40b809f87248d9ec9114bef9)

<img width="1918" height="843" alt="SCREEN10" src="https://github.com/user-attachments/assets/d1e0acac-1bfc-449c-9238-a8a89890437d" />

**Answer:** `BlackSun.ps1`

---

### A ransomware note was saved to disk, which can serve as an IOC. What is the full path to which the ransom note was saved?

1. Query `index=* .txt`

<img width="1642" height="551" alt="SCREEN11" src="https://github.com/user-attachments/assets/53ed5a65-56ae-4aad-ae3c-af01ab41de81" />

**Answer:** `C:\Users\keegan\Downloads\vasg6b0wmw029hd\BlackSun_README.txt`

---

### The script saved an image file to disk to replace the user's desktop wallpaper, which can also serve as an IOC. What is the full path of the image?

1. Query `index=* blacksun`

<img width="1639" height="550" alt="SCREEN12" src="https://github.com/user-attachments/assets/675fd8e5-b1cf-4ab8-824e-c3f2d0d3b63a" />

**Answer:** `C:\Users\Public\Pictures\blacksun.jpg`
