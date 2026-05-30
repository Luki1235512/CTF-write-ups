# [Support](https://tryhackme.com/room/support)

## Pentest the Support Ops platform to exploit vulnerabilities and achieve RCE.

# Support Challenge

A new internal **Support Operations Platform** has been deployed to assist IT and helpdesk teams. The application handles user management, internal APIs, and system-level operations. However, security was not the primary focus during development. Several features rely on user-controlled input and weak trust boundaries.

_Can you pentest the platform and escalate your access to achieve RCE on the server?_

### What is the flag value after logging in as admin?

1. Perform directory enumeration to discover hidden files and directories

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php
```

**Results:**

```
index.php            (Status: 200) [Size: 2591]
info.php             (Status: 200) [Size: 73320]
footer.php           (Status: 200) [Size: 1253]
skins                (Status: 301) [Size: 316] [--> http://<TARGET_IP>/skins/]
includes             (Status: 301) [Size: 319] [--> http://<TARGET_IP>/includes/]
layout               (Status: 301) [Size: 317] [--> http://<TARGET_IP>/layout/]
js                   (Status: 301) [Size: 313] [--> http://<TARGET_IP>/js/]
api.php              (Status: 302) [Size: 0] [--> index.php]
logout.php           (Status: 302) [Size: 0] [--> index.php]
config.php           (Status: 200) [Size: 0]
dashboard.php        (Status: 302) [Size: 0] [--> index.php]
server-status        (Status: 403) [Size: 279]
```

2. Visit h`ttp://<TARGET_IP>/info.php`. This is the standard PHP info page, which leaks detailed server configuration. Notable session-related settings are:

```
session.save_path       = /var/lib/php/sessions
session.use_strict_mode = Off
session.sid_length      = 26
```

3. Brute-force the login page using `hydra`. The helpdesk email `help@support.thm` is visible on the login page and serves as the username

```bash
hydra -l help@support.thm -P /usr/share/wordlists/rockyou.txt <TARGET_IP> http-post-form "/:email=^USER^&password=^PASS^:Invalid credentials" -f
```

4. After logging in, inspect the cookies using the browser's DevTools or Burp Suite. There is a cookie named `isITUser` with the value `68934a3e9455fa72420237eb05902327`. Decoding this MD5 hash reveals the string `false`

5. Encode the string `true` to MD5 and replace the cookie value with the resulting hash

6. Clicking the **IT Admin Panel** button redirects to `api.php`, which documents the available REST API endpoints. The documentation shows that user information can be retrieved with a `GET /user/{id}` call

[SCREEN01]

7. Enumerate users through the API by querying `http://<TARGET_IP>/api.php/user/1`

```json
{
  "email": "specialadmin@support.thm",
  "2FA": false,
  "admin": true
}
```

8. The gobuster results showed that `config.php` exists but returns an empty body, meaning it is a PHP file that only defines variables and produces no output. The `skin` parameter on `dashboard.php` is vulnerable to **Local File Inclusion** via path traversal. The application appends `.php` to the value and includes the resulting file. Using a path traversal payload strips one directory and targets `config.php` directly. View the raw PHP source by prepending `view-source`: `view-source:http://<TARGET_IP>/dashboard.php?skin=../config`

```php
$MASTER_PASSWORD = 'support@110';

$SITE_VER = '1.0';
$SITE_NAME = 'support_portal';
```

9. Log in as `specialadmin@support.thm` using the master password `support110` discovered in the config file. The first flag is displayed on the admin dashboard:

[SCREEN02]

---

### What is the content of the file /home/ubuntu/user.txt?

1. While logged in as admin, the dashboard contains a **Date / Time** widget with a dropdown selector. Intercept the form submission with Burp Suite. The widget sends a POST request to `/dashboard.php` with a `sys` parameter that contains the command passed to the system shell to fetch the current date

2. The `sys` parameter is passed directly to a shell function without any sanitisation, making it vulnerable to **OS command injection**. Append the pipe character `|` to chain an additional shell command after the date call. Send the following POST request to `/dashboard.php`:

[SCREEN03]
