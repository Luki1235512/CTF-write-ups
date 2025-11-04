# [Recovery](https://tryhackme.com/room/recovery)

## Not your conventional CTF

# Help Alex!

Hi, it's me, your friend Alex.

I'm not going to beat around the bush here; I need your help. As you know I work at a company called Recoverysoft. I work on the website side of things, and I setup a Ubuntu web server to run it. Yesterday one of my work colleagues sent me the following email:

```
Hi Alex,
A recent security vulnerability has been discovered that affects the web server. Could you please run this binary on the server to implement the fix?
Regards
- Teo
```

Attached was a linux binary called fixutil. As instructed, I ran the binary, and all was good. But this morning, I tried to log into the server via SSH and I received this message:

```
YOU DIDN'T SAY THE MAGIC WORD!
YOU DIDN'T SAY THE MAGIC WORD!
YOU DIDN'T SAY THE MAGIC WORD!
```

It turns out that Teo got his mail account hacked, and fixutil was a targeted malware binary specifically built to destroy my webserver!

when I opened the website in my browser I get some crazy nonsense. The webserver files had been encrypted! Before you ask, I don't have any other backups of the webserver (I know, I know, horrible practice, etc...), I don't want to tell my boss, he'll fire me for sure.

**Please access the web server and repair all the damage caused by fixutil. You can find the binary in my home directory. Here are my ssh credentials:**

```
Username: alex
Password: madeline
```

**I have setup a control panel to track your progress on port 1337**. Access it via your web browser. As you repair the damage, you can refresh the page to receive those "flags" I know you love hoarding.

Good luck!

\- Your friend Alex

---

### Flag 0

1. Bypass the malicious `.bashrc` by starting a bash shell without loading profile configurations:

```bash
ssh alex@<TARGET_IP> -t '/bin/bash --norc --noprofile'
```

The `-t` flag forces pseudo-terminal allocation, while `--norc` and `--noprofile` prevent bash from reading the `.bashrc` and `.profile` files.

2. List all files including hidden ones to locate the `.bashrc` file:

```bash
ls -la
```

3. Edit the `.bashrc` file to remove the malicious code:

```bash
nano .bashrc
```

Look for suspicious command at the end of the file and remove it

<img width="1012" height="553" alt="SCREEN02" src="https://github.com/user-attachments/assets/6ccf6308-43ea-4884-84d4-d2321a7c4554" />

<img width="997" height="282" alt="SCREEN01" src="https://github.com/user-attachments/assets/54132c05-18bf-4967-a59f-59b2b59045ca" />

4. Refresh `http://<TARGET_IP>:1337/` in your browser to receive **Flag 0**.

---

### Flag 1

1. Navigate to the cron directory to inspect scheduled tasks:

```bash
cd /etc/cron.d
ls -la
```

2. Check the contents of the suspicious cron job:

```bash
cat evil
```

This will reveal a cron job that executes `/opt/brilliant_script.sh` at regular intervals.

<img width="512" height="211" alt="SCREEN03" src="https://github.com/user-attachments/assets/7667607e-f8d7-4a89-8fcb-8d9fc12f75c0" />

3. Navigate to the script location and examine it:

```bash
cd /opt
ls -la
nano brilliant_script.sh
```

4. Remove the malicious command from the script.

<img width="1008" height="101" alt="SCREEN04" src="https://github.com/user-attachments/assets/f57ece4d-ccbe-4f5d-b107-b0394df6cb0d" />

5. Refresh `http://<TARGET_IP>:1337/` to receive **Flag 1**.

---

### Flag 2

1. Download the malicious binary for analysis:

```bash
scp alex@<TARGET_IP>:/home/alex/fixutil .
```

2. Open `fixutil` in Ghidra for reverse engineering:

   - Import the binary into Ghidra
   - Analyze the binary
   - Navigate to the `main` function in the Symbol Tree

3. In the `main` function decompilation, observe references to `liblogging.so` in `/lib/x86_64-linux-gnu/`. This indicates the malware manipulated this library.

<img width="1920" height="1040" alt="SCREEN06" src="https://github.com/user-attachments/assets/8e8c85a5-1325-471a-bba7-1faa6d2ba676" />

4. Download the suspicious library for analysis:

```bash
scp alex@<TARGET_IP>:/lib/x86_64-linux-gnu/liblogging.so .
```

5. Open `liblogging.so` in Ghidra and examine the `LogIncorrectAttempt` function. The decompiled code reveals:
   - The malware copies `/tmp/logging.so` to `/lib/x86_64-linux-gnu/oldliblogging.so`
   - The original legitimate library was backed up as `oldliblogging.so`

<img width="1920" height="900" alt="SCREEN07" src="https://github.com/user-attachments/assets/9f8eb2de-55a3-4d2c-b873-a6b2d59427b9" />

6. To restore the library, we need sudo privileges. Add a sudo entry via the `brilliant_script.sh` that runs as root:

```bash
echo 'echo "alex ALL=(ALL:ALL) ALL" >> /etc/sudoers;' >> /opt/brilliant_script.sh
cat brilliant_script.sh
```

7. Wait for the cron job to execute, then verify sudo access:

```bash
sudo -l
```

<img width="409" height="66" alt="SCREEN05" src="https://github.com/user-attachments/assets/dbd39bd4-7d41-428c-92f0-785bd7795b02" />

8. Restore the original library by renaming the backup:

```bash
sudo mv oldliblogging.so liblogging.so
```

9. Refresh `http://<TARGET_IP>:1337/` to receive **Flag 2**.

---

### Flag 3

1. The malware added attacker-controlled SSH keys to `/root/.ssh/authorized_keys`, allowing password-less authentication.

2. Remove the unauthorized SSH keys:

```bash
sudo rm /root/.ssh/authorized_keys
```

This prevents the attacker from accessing the root account via SSH using their private key.

3. Refresh `http://<TARGET_IP>:1337/` to receive **Flag 3**.

---

### Flag 4

1. User account are defined in `/etc/passwd` and their password hashes in `/etc/shadow`. Remove the "security" user from both files.

```bash
sudo nano /etc/passwd
sudo nano /etc/shadow
```

<img width="1008" height="559" alt="SCREEN08" src="https://github.com/user-attachments/assets/810b7d4d-dc0b-4f36-970a-d9e1637a821f" />

2. Refresh `http://<TARGET_IP>:1337/` to receive **Flag 4**.

---

### Flag 5

1. In Ghidra, examine the `XOREncryptWebFiles` function in the `fixutil` binary:
   - It generates a random string using `rand_string` function
   - Stores the encryption key in `/opt/.fixutil/backup.txt`
   - Calls `GetWebFiles` to find all files in the web directory
   - XORs each file using the `XORFile` function

<img width="750" height="666" alt="SCREEN09" src="https://github.com/user-attachments/assets/1681ce74-652b-44d6-bb8c-cf7f65e4c783" />

Here it first creates a random string using `rand_string` function and stores the string is `/opt/.fixutil/backup.txt` file. Then finds the webfiles using `GetWebFiles` function and XOR’s the content using `XORFile` function.

2. Examine the `GetWebFiles` function:
   - Opens the directory defined in `web_location` variable
   - Returns all files in `/usr/local/apache2/htdocs` directory
   - These are the files that need to be decrypted

<img width="1644" height="641" alt="SCREEN10" src="https://github.com/user-attachments/assets/a185f915-37b9-4b05-9ebc-21fa24733872" />

3. Examine the `XORFile` function:
   - Reads each file in binary mode
   - XORs each byte with the corresponding byte from the encryption key
   - The key is used in a repeating pattern (key[i % key_length])

<img width="751" height="740" alt="SCREEN11" src="https://github.com/user-attachments/assets/c5017d84-fb25-4484-8c68-a4540e386c40" />

Reads the file in binary format, and XORs the file with the encryption key

4. Download the encrypted web files:

```bash
scp alex@<TARGET_IP>:/usr/local/apache2/htdocs/* .
```

5. Retrieve the encryption key:

```bash
sudo cat /opt/.fixutil/backup.txt
```

<img width="647" height="32" alt="SCREEN12" src="https://github.com/user-attachments/assets/a6546813-d4cf-4104-bfb5-816df25ccd26" />

6. Create a Python script to decrypt the files:

```python
key = b"AdsipPewFlfkmll"
files = ["index.html", "reallyimportant.txt", "todo.html"]

for fil in files:
    with open(fil, "rb") as f:
        contents = f.read()

    decrypted = ""
    for i in range(len(contents)):
        decrypted += chr(contents[i] ^ key[i % len(key)])

    with open(fil, "w") as f:
        f.write(decrypted)
```

7. Run the decryption script:

```bash
python3 script.py
```

8. Upload the decrypted files back to the server:

```bash
scp ./* alex@<TARGET_IP>:/home/alex
```

9. Move the decrypted files to the web directory:

```bash
cd /usr/local/apache2/htdocs/
sudo mv /home/alex/{index.html,reallyimportant.txt,todo.html} .
```

<img width="904" height="113" alt="SCREEN13" src="https://github.com/user-attachments/assets/3d5fdcd5-c18a-43eb-aef6-c4e665f36cc9" />

11. Refresh `http://<TARGET_IP>:1337/` to receive **Flag 5**.
