# [Scripting](https://tryhackme.com/room/scripting)

## Learn basic scripting by solving some challenges!

# [Easy] Base64

## This file has been base64 encoded 50 times - write a script to retrieve the flag. Try do this in both Bash and Python!

### What is the final string?

1. Python script:

```Python
import base64

with open('b64_1550406728131.txt') as f:
    msg = f.read()
    for _ in range(50):
        msg = base64.b64decode(msg)
    print(f"The flag is: {msg.decode('utf8')}")
```

Remember make the .sh script executable

```Bash
chmod +x decode_base64.sh
```

```Shell
#!/bin/bash
msg=$(cat ${1:-b64_1550406728131.txt})
for i in {1..50}; do msg=$(echo -n "$msg" | base64 -d); done
echo "The flag is: $msg"
```

---

# [Medium] Gotta Catch em All

## You need to write a script that connects to this webserver on the correct port, do an operation on a number and then move onto the next port. Start your original number at 0.

### Once you have done all operations, what number do you get (rounded to 2 decimal places at the end of your calculation)?

```Python
import socket
import time
import re
import sys

def main():
    server_ip, port, old_num = sys.argv[1], 1337, 0

    while port != 9765:
        try:
            with socket.socket() as s:
                s.connect((server_ip, port))
                s.send(f"GET / HTTP/1.0\r\nHost: {server_ip}:{port}\r\n\r\n".encode())

                data = ""
                while True:
                    response = s.recv(1024)
                    if not response:
                        break
                    data = response.decode()

                # Extract operation, number, and next port
                parts = re.split(r'[ *\n]+', data)
                parts = [p for p in parts if p]
                op, new_num, port = parts[-3], float(parts[-2]), int(parts[-1])

                # Perform calculation
                operations = {'add': lambda x, y: x + y, 'minus': lambda x, y: x - y,
                            'divide': lambda x, y: x / y, 'multiply': lambda x, y: x * y}
                old_num = operations[op](old_num, new_num)

                print(f"Current number: {old_num}, next port: {port}")

        except:
            print(f"Waiting for port {port}...")
            time.sleep(3)

    print(f"Final answer: {round(old_num, 2)}")

if __name__ == '__main__':
    main()
```

---

# [Hard] Encrypted Server Chit Chat

## The VM you have to connect to has a UDP server running on port 4000. Once connected to this UDP server, send a UDP message with the payload "hello" to receive more information. You will find some sort of encryption(using the AES-GCM cipher). Using the information from the server, write a script to retrieve the flag

### What is the flag?

```Python
import socket, hashlib, sys
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend

def Main():
    host, port = sys.argv[1], 4000
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    server = (host, port)

    # Initial handshake
    s.sendto(b"hello", server)
    print(s.recv(1024))
    s.sendto(b"ready", server)
    data = s.recv(1024)
    checksum = data[104:136].hex()

    # Decrypt loop
    while True:
        s.sendto(b"final", server)
        cText = s.recv(1024)
        s.sendto(b"final", server)
        tag = s.recv(1024)

        # Decrypt with AES-GCM
        decryptor = Cipher(algorithms.AES(b'thisisaverysecretkeyl337'),
                          modes.GCM(b'secureivl337', tag),
                          backend=default_backend()).decryptor()
        pText = decryptor.update(cText) + decryptor.finalize()

        if hashlib.sha256(pText).hexdigest() == checksum:
            print(f"Flag: {pText}")
            break

if __name__ == '__main__': Main()
```
