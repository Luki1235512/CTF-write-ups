# [Biohazard](https://tryhackme.com/room/biohazard)

## A CTF room based on the old-time survival horror game, Resident Evil. Can you survive until the end?

# Introduction

## Welcome to Biohazard room, a puzzle-style CTF. Collecting the item, solving the puzzle and escaping the nightmare is your top priority. Can you survive until the end?

### How many open ports?

1. First, let's scan the target machine to identify open ports:

```Bash
nmap IP
```

The scan reveals **3 open ports**:

- Port 21: FTP service
- Port 22: SSH service
- Port 80: HTTP web server

![SCREEN01](https://github.com/user-attachments/assets/df4465d2-78b8-4397-a3c9-351bc8a8edd2)

### What is the team name in operation?

1. Navigate to the main web page at http://IP/
2. Examine the page content - the team name appears directly below the main image
3. The special forces team operating in this scenario is **STARS alpha team**

![SCREEN02](https://github.com/user-attachments/assets/0ceb66a4-d0a0-4ef1-b5d7-9b0269ef756f)

# The Mansion

## Collect all necessary items and advanced to the next level. The format of the Item flag: Item_name{32 character}. Some of the doors are locked. Use the item flag to unlock the door.

### What is the emblem flag

1. From the main page, navigate to `http://IP/mansionmain/`

   - Inspect the page source to find hidden comments
   - The comments contain a clue directing us to `/diningRoom/`

2. Navigate to `http://IP/diningRoom/`
   - Here you'll find a link to `emblem.php`
   - Visit `http://IP/diningRoom/emblem.php` to obtain the **emblem flag**

![SCREEN03](https://github.com/user-attachments/assets/e6282461-12af-4598-9203-ce4fc6ee61a4)

### What is the lock pick flag

1. Examine the source code of the dining room page (`http://IP/diningRoom/`)

   - Find the Base64 encoded string: `SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=`

2. Decode this using Base64 in [CyberChef](https://gchq.github.io/CyberChef/)
   - The decoded message reads: `How about the /teaRoom/`

![SCREEN04](https://github.com/user-attachments/assets/0a77b620-c3e4-46dd-8d49-5ef6744cf541)

3. Follow the hint to `http://IP/teaRoom/`
   - Find a link to `master_of_unlock.html`
   - Visit `http://IP/teaRoom/master_of_unlock.html` to obtain the **lock pick flag**

![SCREEN05](https://github.com/user-attachments/assets/b5ec23b5-a84d-43f8-aea7-b6cbacc05559)

### What is the music sheet flag

1. Read the text in the `/teaRoom/` page

   - It guides you to the `/artRoom/`
   - In the art room, find a link to `/artRoom/MansionPage.html`

2. From `http://IP/artRoom/MansionMap.html`

   - Locate the link to `/barRoom/`
   - You'll need to use the lock pick flag to open this door

3. After successfully using the lock pick, you'll arrive at:

   - `http://IP/barRoom357162e3db904857963e6e0b64b96ba7/`
   - Find a link to a music note: `musicNote.html`

4. The music note contains encoded text
   - Decode this using Base32 in [CyberChef](https://gchq.github.io/CyberChef/)
   - This reveals the **music sheet flag**

![SCREEN06](https://github.com/user-attachments/assets/2fd9f06d-13c4-4a0f-93b1-f29b48133f02)

### What is the gold emblem flag

1. "Play" the music sheet (submit the flag) at:
   - `http://IP/barRoom357162e3db904857963e6e0b64b96ba7/`
   - This will redirect you to `barRoomHidden.php`
   - Here you'll find a link to obtain the **gold emblem flag**

![SCREEN07](https://github.com/user-attachments/assets/965d73ec-2be2-4958-8659-a5c7629deafc)

### What is the shield key flag

1. In `http://IP/barRoom357162e3db904857963e6e0b64b96ba7/barRoomHidden.php`:

   - Submit the original emblem flag to access `emblem_slot.php`
   - This reveals the name `rebecca`

2. Return to `http://IP/diningRoom/`:

   - Submit the gold emblem flag to the emblem slot
   - This provides you with a Vigenère cipher text

3. Decrypt the Vigenère cipher:
   - Use [CyberChef](https://gchq.github.io/CyberChef/) with `rebecca` as the key
   - The decrypted message contains a URL

![SCREEN08](https://github.com/user-attachments/assets/c520f2a4-d70a-4670-b0d4-afffac1f02bd)

4. Follow the decrypted link to:
   - `http://IP/diningRoom/the_great_shield_key.html`
   - Here you'll obtain the **shield key flag**

![SCREEN09](https://github.com/user-attachments/assets/54eeac99-15f0-408d-8340-3951901d4cdd)

### What is the blue gem flag

1. Inspect the source code of `http://IP/diningRoom2F/`
   - Discover a ROT13 encrypted message

![SCREEN10](https://github.com/user-attachments/assets/f4168f27-d949-48ce-bf59-f5da9cde6c1f)

2. Decrypt the ROT13 message:
   - This will lead you to `http://IP/diningRoom/sapphire.html`
   - Here you'll find the **blue gem flag**

![SCREEN11](https://github.com/user-attachments/assets/142f05a0-0712-4a4b-b64d-036bc1dd104b)

### What is the FTP username

### What is the FTP password

To obtain FTP credentials, you'll need to collect and decode four crests:

1. First crest:
   - Visit `http://IP/tigerStatusRoom/`
   - Submit the blue gem flag to `gem.php`
   - This reveals the first encoded crest

![SCREEN12](https://github.com/user-attachments/assets/e51a4faf-d3fb-4db0-9fdb-e213de976926)

2. Decode the first crest:
   - Use [CyberChef](https://gchq.github.io/CyberChef/) with Base64 → Base32

![SCREEN13](https://github.com/user-attachments/assets/97cb34be-984c-4f82-8bd8-20ceda20e147)

3. Find the second crest:

   - Navigate to `http://IP/galleryRoom/note.txt`

4. Decode the second crest:

   - Use [CyberChef](https://gchq.github.io/CyberChef/) with Base32 → Base58

![SCREEN14](https://github.com/user-attachments/assets/3123d44a-e80b-4b21-8f61-ecc9e1fee1d4)

5. For the third crest:

   - Go to `http://IP/armorRoom/`
   - Use the shield key flag to unlock the door
   - Access `http://IP/armorRoom547845982c18936a25a9b37096b21fc1/`

6. Decode the third crest:

   - Use [CyberChef](https://gchq.github.io/CyberChef/) with Base64 → Binary → Hex

![SCREEN15](https://github.com/user-attachments/assets/de41cfa9-b3d8-4955-b757-de9dde13b363)

7. For the final crest:

   - Visit `http://IP/attic/`
   - Use the shield key flag to unlock
   - Access `http://IP/attic909447f184afdfb352af8b8a25ffff1d/note.txt`

8. Decode the fourth crest:

   - Use [CyberChef](https://gchq.github.io/CyberChef/) with Base58 → Hex

![SCREEN16](https://github.com/user-attachments/assets/beb2412b-b566-49ee-ad20-b5a3147d5bdb)

9. Combine all four decoded crest fragments:

   - Concatenate them in order
   - Decode the combined string with Base64
   - This reveals the **FTP username and password**

![SCREEN17](https://github.com/user-attachments/assets/d329a390-3540-404a-a980-641710aa417a)

# The guard house

## After gaining access to the FTP server, you need to solve another puzzle.

### Where is the hidden directory mentioned by Barry

1. Login to the FTP server using the credentials we obtained, and download all the files:

```Bash
ftp IP 21
mget 001-key.jpg 002-key.jpg 003-key.jpg helmet_key.txt.gpg important.txt
```

<!-- 2. The `important.txt` file mentions `/hidden_closet/` -->

2. Examine the `important.txt` file contents:

   - The note mentions a hidden directory: `/hidden_closet/`

![SCREEN18](https://github.com/user-attachments/assets/8ff4c0dd-89fd-4a77-b641-30cbc892fd68)

### Password for the encrypted file

1. We need to extract hidden data from the three image files using steghide, exiftool, and binwalk:

```Bash
steghide extract -sf 001-key.jpg
exiftool 002-key.jpg
binwalk -e 003-key.jpg
```

(If you're using the TryHackMe attack-box and binwalk is not working correctly, fix it with:)

```Bash
sed -i 's/CS_ARCH_ARM64/CS_ARCH_AARCH64/g' /usr/lib/python3/dist-packages/binwalk/modules/disasm.py
```

![SCREEN19](https://github.com/user-attachments/assets/66c85f77-69dc-42c5-a20b-fdcbb72b3ecf)
![SCREEN20](https://github.com/user-attachments/assets/f39ff16a-517b-4126-9fd6-e134b1c244f1)
![SCREEN21](https://github.com/user-attachments/assets/2d239676-a331-4902-96e6-f6fd95c80e26)

2. From each image, we get a piece of the password. Combine all three parts, and decode the resulting string with Base64 to get the final password

### What is the helmet key flag

1. Use the password to decrypt the GPG-encrypted file

![SCREEN23](https://github.com/user-attachments/assets/cb6644c9-2aa1-4de2-b716-72a09a752351)

# The revisit

## Done with the puzzle? There are places you have explored before but yet to access.

### What is the SSH login username

1. With our helmet flag we can access `/studyRoom/`

   - Submit the helmet key to unlock access
   - This redirects to `http://IP/studyRoom28341c5e98c93b89258a6389fd608a3c/`
   - Download `doom.tar.gz` file
   - Inside, find `eagle_medal.txt` with SSH username: **umbrella_guest**

![SCREEN24](https://github.com/user-attachments/assets/feb3cf0c-f9b3-4740-a4b4-bff1adaea38d)

### What is the SSH login password

1. Use the helmet flag to access the hidden closet:

   - Navigate to `/hidden_closet/`
   - Submit the helmet key to unlock access
   - You'll be redirected to `http://IP/hiddenCloset8997e740cb7f5cece994381b9477ec38/`
   - Find `wolf_medal.txt`
   - This file contains the SSH password: **T_virus_rules**

![SCREEN26](https://github.com/user-attachments/assets/7a9a5533-2171-4b6c-92d0-1056bb913301)

### Who the STARS bravo team leader

1. The information is found in the hidden closet text:

   - Read the narrative text at `http://IP/hiddenCloset8997e740cb7f5cece994381b9477ec38/`
   - The STARS bravo team leader is **Enrico**

![SCREEN25](https://github.com/user-attachments/assets/8c264cae-d602-43eb-9a1e-f580597f1ac0)

# Underground laboratory

## Time for the final showdown. Can you escape the nightmare?

### Where you found Chris

### Who is the traitor

1. Login to the machine using SSH with the credentials we've gathered:

   - Username: umbrella_guest
   - Password: T_virus_rules

2. Explore the file system to locate Chris:
   - Check for hidden files and directories with `ls -a`
   - Navigate to the hidden jailcell directory: `cd .jailcell/`
   - Read the content with: `cat chris.txt`
   - Inside you'll find information about Chris and the traitor
   - Chris was found in the **jailcell** and the traitor is **Wesker**

```Bash
ssh umbrella_guest@IP
ls -a
cd .jailcell/
ls
cat chris.txt
```

![SCREEN27](https://github.com/user-attachments/assets/c83ba177-468d-45f4-886b-086a562cee7a)

### The login password for the traitor

1. Return to the web interface:

   - Navigate to the hidden closet at `http://IP/hiddenCloset8997e740cb7f5cece994381b9477ec38/`
   - Find and open `MO_DISK1.txt`

2. The disk contains a Vigenère cipher:
   - Use [CyberChef](https://gchq.github.io/CyberChef/) to decode it
   - Apply key: `albert`
   - The decrypted text reveals Wesker's password: **stars_members_are_my_guinea_pig**

![SCREEN28](https://github.com/user-attachments/assets/0ee46aed-105e-41fa-8a1f-c7bcb26b8272)

### The name of the ultimate form

1. Login as Wesker using SSH:

2. Examine Wesker's files:
   - List files with `ls`
   - Read his note: `cat wesker_note.txt`
   - The ultimate form is revealed as **Tyrant**

```Bash
ssh weasker@IP
ls
cat weasker_note.txt
```

![SCREEN29](https://github.com/user-attachments/assets/8d129f38-8ed5-4e86-ae0d-031b870e526a)

### The root flag

1. Wesker has full sudo privileges, so we can access the root flag:

```Bash
sudo -l
sudo cat /root/root.txt
```

![SCREEN30](https://github.com/user-attachments/assets/0bbf7716-ffbd-488f-af32-5e69d6a6ed0e)
